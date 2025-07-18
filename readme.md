import os
import threading
import time
import json
import re
from datetime import datetime, timedelta
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import google.generativeai as genai
from dotenv import load_dotenv
from typing import Optional, Dict, Any

# Load environment variables
load_dotenv()
GEMINI_API_KEY = os.getenv('GOOGLE_API_KEY')
LLM_OUTPUT_FILE = "llm_definitions.jsonl"
ACTION_ITEMS_FILE = "action_items.jsonl"

# Real-time configuration
REALTIME_CONFIG = {
    'LLM_INTERVAL': int(os.getenv('LLM_INTERVAL', '2')),  # 2 seconds for real-time
    'TRANSCRIPTION_WINDOW': int(os.getenv('TRANSCRIPTION_WINDOW', '8')),  # 8 seconds
    'ACTIVITY_BOOST': float(os.getenv('ACTIVITY_BOOST', '0.5')),  # 50% faster when active
    'QUIET_THRESHOLD': int(os.getenv('QUIET_THRESHOLD', '5')),  # 5 seconds quiet threshold
}

# FastAPI app
app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

genai.configure(api_key=GEMINI_API_KEY)

# Globals for background tasks
llm_thread = None
llm_running = threading.Event()

# Shared data with activity tracking
class SharedState:
    def __init__(self):
        self.transcription_buffer = []  # list of (timestamp, text, is_active)
        self.llm_output = None
        self.action_items = []  # Store recent action items
        self.lock = threading.Lock()
        self.last_activity = datetime.utcnow()
        self.activity_level = 0  # 0-1 scale
        self.config = REALTIME_CONFIG.copy()

    def add_transcription(self, text, is_active=False):
        with self.lock:
            now = datetime.utcnow()
            self.transcription_buffer.append((now, text, is_active))
            
            if is_active:
                self.last_activity = now
                self.activity_level = min(1.0, self.activity_level + 0.2)
            else:
                # Decay activity level
                time_since_activity = (now - self.last_activity).total_seconds()
                if time_since_activity > self.config['QUIET_THRESHOLD']:
                    self.activity_level = max(0.0, self.activity_level - 0.1)
            
            # Keep only last N seconds based on activity
            window_size = self.config['TRANSCRIPTION_WINDOW']
            if self.activity_level > 0.5:
                window_size = int(window_size * 1.5)  # Longer window when active
            
            cutoff = now - timedelta(seconds=window_size)
            self.transcription_buffer = [x for x in self.transcription_buffer if x[0] >= cutoff]

    def get_last_n_seconds(self, seconds=None):
        with self.lock:
            if seconds is None:
                seconds = self.config['TRANSCRIPTION_WINDOW']
            
            cutoff = datetime.utcnow() - timedelta(seconds=seconds)
            return ' '.join([t for ts, t, _ in self.transcription_buffer if ts >= cutoff])

    def get_dynamic_interval(self):
        with self.lock:
            base_interval = self.config['LLM_INTERVAL']
            if self.activity_level > 0.5:
                return max(1, int(base_interval * self.config['ACTIVITY_BOOST']))
            else:
                return base_interval * 2  # Slower when quiet

    def set_llm_output(self, output):
        with self.lock:
            self.llm_output = output

    def get_llm_output(self):
        with self.lock:
            return self.llm_output

    def add_action_item(self, action_item):
        with self.lock:
            self.action_items.append(action_item)
            # Keep only last 50 action items
            if len(self.action_items) > 50:
                self.action_items = self.action_items[-50:]

    def get_recent_action_items(self, limit=10):
        with self.lock:
            return self.action_items[-limit:]

    def update_config(self, new_config):
        with self.lock:
            self.config.update(new_config)

shared_state = SharedState()

# Technical definitions schema (existing)
TECHNICAL_DEFINITIONS_SCHEMA = '''
{
  "type": "object",
  "properties": {
    "technical_terms": {
      "type": "array",
      "description": "A list of all technical terms mentioned in the transcript, along with definitions and contextual explanations.",
      "items": {
        "type": "object",
        "properties": {
          "term": {"type": "string", "description": "The technical term or jargon identified in the transcript."},
          "definition": {"type": "string", "description": "A clear and concise definition of the technical term."},
          "contextual_explanation": {"type": "string", "description": "An explanation of how the term is used or meant within the current transcript, tailored for a non-technical audience."},
          "example_quote": {"type": "string", "description": "A direct quote or sentence from the transcript where the term appears."}
        },
        "required": ["term", "definition", "contextual_explanation"]
      }
    }
  },
  "required": ["technical_terms"]
}
'''

# Action items detection and generation schemas
ACTION_DETECTION_SCHEMA = '''
{
  "type": "object",
  "properties": {
    "has_action_items": {
      "type": "boolean",
      "description": "Whether the transcript contains any action-oriented phrases that require note-taking or action items."
    },
    "action_triggers": {
      "type": "array",
      "description": "List of phrases or contexts that triggered action item detection.",
      "items": {
        "type": "object",
        "properties": {
          "trigger_phrase": {"type": "string", "description": "The phrase that triggered action detection (e.g., 'note this down', 'create a form')."},
          "context": {"type": "string", "description": "The surrounding context where the trigger appeared."},
          "action_type": {"type": "string", "description": "Type of action needed (note, form, task, reminder, etc.)"}
        },
        "required": ["trigger_phrase", "context", "action_type"]
      }
    }
  },
  "required": ["has_action_items", "action_triggers"]
}
'''

ACTION_GENERATION_SCHEMA = '''
{
  "type": "object",
  "properties": {
    "action_items": {
      "type": "array",
      "description": "Generated action items based on the transcript context.",
      "items": {
        "type": "object",
        "properties": {
          "title": {"type": "string", "description": "A clear, concise title for the action item."},
          "description": {"type": "string", "description": "Detailed description of what needs to be done."},
          "type": {"type": "string", "description": "Type of action (note, form, task, reminder, follow-up, etc.)"},
          "priority": {"type": "string", "description": "Priority level (high, medium, low)."},
          "context": {"type": "string", "description": "The original context from the transcript that led to this action."},
          "suggested_content": {"type": "string", "description": "If applicable, suggested content for the note/form/task."}
        },
        "required": ["title", "description", "type", "priority", "context"]
      }
    }
  },
  "required": ["action_items"]
}
'''

def get_gemini_definitions(text):
    prompt = (
        "Extract all technical terms from the following text and provide their definitions and context. "
        "Return the result as a JSON object matching this schema (if no terms, use an empty list for 'technical_terms'):\n" + TECHNICAL_DEFINITIONS_SCHEMA + "\nText:\n" + text
    )
    model = genai.GenerativeModel('gemini-2.0-flash')
    response = model.generate_content(prompt)
    match = re.search(r'\{[\s\S]*\}', response.text)
    if match:
        json_str = match.group(0)
        try:
            return json.loads(json_str)
        except Exception as e:
            print(f"[DEFINITIONS JSON ERROR] {e}\nRaw output: {json_str}")
            return None
    else:
        print(f"[DEFINITIONS NO JSON FOUND] Raw output: {response.text}")
        return None

def detect_action_items(text):
    """Detect if the text contains action-oriented phrases"""
    prompt = (
        "Analyze the following transcript and detect if it contains any action-oriented phrases that would require the listener to take notes, create forms, start tasks, or perform any other actions. "
        "Look for phrases like: 'note this down', 'create a note', 'write this down', 'remember this', 'action item', 'to-do', 'create a form', 'fill this form', 'start a task', 'follow up on', 'schedule', 'reminder', etc. "
        "Return the result as a JSON object matching this schema:\n" + ACTION_DETECTION_SCHEMA + "\nTranscript:\n" + text
    )
    model = genai.GenerativeModel('gemini-2.0-flash')
    response = model.generate_content(prompt)
    match = re.search(r'\{[\s\S]*\}', response.text)
    if match:
        json_str = match.group(0)
        try:
            return json.loads(json_str)
        except Exception as e:
            print(f"[ACTION DETECTION JSON ERROR] {e}\nRaw output: {json_str}")
            return None
    else:
        print(f"[ACTION DETECTION NO JSON FOUND] Raw output: {response.text}")
        return None

def generate_action_items(text, detected_actions):
    """Generate specific action items based on the detected actions"""
    prompt = (
        "Based on the following transcript and detected action triggers, generate specific, actionable items that the listener should note down or act upon. "
        "Make the action items clear, specific, and actionable. Consider the context and generate appropriate notes, forms, tasks, or reminders.\n"
        f"Detected actions: {json.dumps(detected_actions, indent=2)}\n"
        "Return the result as a JSON object matching this schema:\n" + ACTION_GENERATION_SCHEMA + "\nTranscript:\n" + text
    )
    model = genai.GenerativeModel('gemini-2.0-flash')
    response = model.generate_content(prompt)
    match = re.search(r'\{[\s\S]*\}', response.text)
    if match:
        json_str = match.group(0)
        try:
            return json.loads(json_str)
        except Exception as e:
            print(f"[ACTION GENERATION JSON ERROR] {e}\nRaw output: {json_str}")
            return None
    else:
        print(f"[ACTION GENERATION NO JSON FOUND] Raw output: {response.text}")
        return None

# Enhanced LLM processor with action item detection
class EnhancedLLMWorker(threading.Thread):
    def __init__(self, shared_state):
        super().__init__(daemon=True)
        self.shared_state = shared_state
        self.running = llm_running

    def run(self):
        while self.running.is_set():
            # Get dynamic interval based on activity
            interval = self.shared_state.get_dynamic_interval()
            
            print(f"[LLM] Sleeping for {interval}s (activity: {self.shared_state.activity_level:.2f})")
            time.sleep(interval)
            
            recent_text = self.shared_state.get_last_n_seconds()
            if recent_text.strip():
                print(f"[LLM] Processing: {recent_text[:100]}...")
                
                # Process technical definitions (existing functionality)
                definitions = get_gemini_definitions(recent_text)
                
                # Process action items (new functionality)
                action_detection = detect_action_items(recent_text)
                action_items_result = None
                
                if action_detection and action_detection.get('has_action_items', False):
                    print(f"[ACTION] Detected action items: {len(action_detection.get('action_triggers', []))}")
                    action_items_result = generate_action_items(recent_text, action_detection)
                    
                    if action_items_result:
                        # Save action items to file
                        action_obj = {
                            "timestamp": datetime.utcnow().isoformat(),
                            "transcript": recent_text,
                            "detection": action_detection,
                            "action_items": action_items_result,
                            "activity_level": self.shared_state.activity_level
                        }
                        with open(ACTION_ITEMS_FILE, 'a', encoding='utf-8') as f:
                            f.write(json.dumps(action_obj) + '\n')
                        
                        # Add to shared state
                        self.shared_state.add_action_item(action_obj)
                
                # Save technical definitions (existing functionality)
                if definitions:
                    obj = {
                        "timestamp": datetime.utcnow().isoformat(),
                        "transcript": recent_text,
                        "llm_output": definitions,
                        "action_detection": action_detection,
                        "action_items": action_items_result,
                        "activity_level": self.shared_state.activity_level,
                        "interval_used": interval
                    }
                    with open(LLM_OUTPUT_FILE, 'a', encoding='utf-8') as f:
                        f.write(json.dumps(obj) + '\n')
                    self.shared_state.set_llm_output(obj)

# API Models
class StatusResponse(BaseModel):
    running: bool
    config: Optional[Dict[str, Any]] = None
    activity_level: Optional[float] = None

class TranscriptionRequest(BaseModel):
    text: str
    timestamp: Optional[int] = None
    is_active: Optional[bool] = False

class SessionConfig(BaseModel):
    config: Optional[Dict[str, Any]] = None

@app.post("/session/start", response_model=StatusResponse)
def start_session(session_config: Optional[SessionConfig] = None):
    global llm_thread
    
    if llm_running.is_set():
        return {
            "running": True, 
            "config": shared_state.config,
            "activity_level": shared_state.activity_level
        }
    
    # Update config if provided
    if session_config and session_config.config:
        shared_state.update_config(session_config.config)
        print(f"[SESSION] Updated config: {session_config.config}")
    
    llm_running.set()
    shared_state.transcription_buffer.clear()
    shared_state.llm_output = None
    shared_state.action_items = []
    shared_state.activity_level = 0
    
    llm_thread = EnhancedLLMWorker(shared_state)
    llm_thread.start()
    
    return {
        "running": True,
        "config": shared_state.config,
        "activity_level": shared_state.activity_level
    }

@app.post("/session/stop", response_model=StatusResponse)
def stop_session():
    llm_running.clear()
    return {
        "running": False,
        "config": shared_state.config,
        "activity_level": shared_state.activity_level
    }

@app.post("/process-transcription")
def process_transcription(request: TranscriptionRequest):
    """Receive transcription from client and add to buffer for LLM processing"""
    shared_state.add_transcription(request.text, request.is_active)
    return {
        "status": "received",
        "activity_level": shared_state.activity_level,
        "next_interval": shared_state.get_dynamic_interval()
    }

@app.get("/llm/latest")
def get_latest_llm():
    output = shared_state.get_llm_output()
    if output is None:
        raise HTTPException(status_code=404, detail="No LLM output yet.")
    return output

@app.get("/action-items/latest")
def get_latest_action_items():
    """Get the most recent action items"""
    action_items = shared_state.get_recent_action_items()
    if not action_items:
        raise HTTPException(status_code=404, detail="No action items yet.")
    return {
        "action_items": action_items,
        "count": len(action_items)
    }

@app.get("/action-items/recent/{limit}")
def get_recent_action_items(limit: int = 10):
    """Get recent action items with a specified limit"""
    action_items = shared_state.get_recent_action_items(limit)
    return {
        "action_items": action_items,
        "count": len(action_items)
    }

@app.get("/session/status")
def get_session_status():
    return {
        "running": llm_running.is_set(),
        "config": shared_state.config,
        "activity_level": shared_state.activity_level,
        "buffer_size": len(shared_state.transcription_buffer),
        "next_interval": shared_state.get_dynamic_interval(),
        "action_items_count": len(shared_state.action_items)
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
