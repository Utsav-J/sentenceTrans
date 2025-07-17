Perfect! I can see your current implementation uses server-side Whisper transcription with FastAPI. Let me help you convert this to client-side Web Speech API transcription while keeping the LLM processing on the server.

Here's your updated React app with client-side speech recognition:

```jsx
import React, { useState, useEffect, useRef } from 'react';
import './App.css';
import { FaMicrophone, FaGithub, FaSun, FaMoon, FaStop } from 'react-icons/fa';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:8000';
const DEFINITIONS_CACHE_KEY = 'wf_teams_definitions_cache';

function App() {
  const [isRecording, setIsRecording] = useState(false);
  const [transcription, setTranscription] = useState('');
  const [interimTranscription, setInterimTranscription] = useState('');
  const [loading, setLoading] = useState(false);
  const [darkMode, setDarkMode] = useState(false);
  const [hasTranscribed, setHasTranscribed] = useState(false);
  const [definitions, setDefinitions] = useState([]);
  const [isSupported, setIsSupported] = useState(false);
  
  const recognitionRef = useRef(null);
  const pollingRef = useRef(null);
  const transcriptionBuffer = useRef([]);
  const lastSentTime = useRef(0);

  // Initialize Speech Recognition
  useEffect(() => {
    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    
    if (SpeechRecognition) {
      setIsSupported(true);
      
      const recognition = new SpeechRecognition();
      recognition.continuous = true;
      recognition.interimResults = true;
      recognition.lang = 'en-US';
      
      recognition.onresult = (event) => {
        let finalTranscript = '';
        let interimTranscript = '';

        for (let i = event.resultIndex; i < event.results.length; i++) {
          const transcript = event.results[i][0].transcript;
          if (event.results[i].isFinal) {
            finalTranscript += transcript;
          } else {
            interimTranscript += transcript;
          }
        }

        if (finalTranscript) {
          setTranscription(prev => {
            const newTranscription = prev + finalTranscript;
            transcriptionBuffer.current.push({
              timestamp: new Date(),
              text: finalTranscript
            });
            return newTranscription;
          });
          setHasTranscribed(true);
        }
        
        setInterimTranscription(interimTranscript);
      };

      recognition.onstart = () => {
        setIsRecording(true);
        setLoading(false);
      };

      recognition.onend = () => {
        setIsRecording(false);
        setInterimTranscription('');
      };

      recognition.onerror = (event) => {
        console.error('Speech recognition error:', event.error);
        setIsRecording(false);
        setLoading(false);
        if (event.error === 'not-allowed') {
          alert('Microphone access denied. Please allow microphone access and try again.');
        }
      };

      recognitionRef.current = recognition;
    }

    return () => {
      if (recognitionRef.current) {
        recognitionRef.current.stop();
      }
    };
  }, []);

  // Load cached definitions on mount
  useEffect(() => {
    const cached = localStorage.getItem(DEFINITIONS_CACHE_KEY);
    if (cached) {
      try {
        setDefinitions(JSON.parse(cached));
      } catch {
        setDefinitions([]);
      }
    }
  }, []);

  useEffect(() => {
    document.body.setAttribute('data-theme', darkMode ? 'dark' : 'light');
  }, [darkMode]);

  // Send transcription to server and poll for LLM output
  useEffect(() => {
    if (isRecording) {
      pollingRef.current = setInterval(async () => {
        // Send recent transcription to server for LLM processing
        const now = Date.now();
        const recentTranscription = getRecentTranscription(5000); // Last 5 seconds
        
        if (recentTranscription && now - lastSentTime.current > 3000) {
          try {
            await fetch(`${API_URL}/process-transcription`, {
              method: 'POST',
              headers: {
                'Content-Type': 'application/json',
              },
              body: JSON.stringify({ text: recentTranscription })
            });
            lastSentTime.current = now;
          } catch (err) {
            console.error('Error sending transcription:', err);
          }
        }

        // Get latest LLM output
        try {
          const res = await fetch(`${API_URL}/llm/latest`);
          if (res.ok) {
            const llmData = await res.json();
            if (llmData.llm_output && llmData.llm_output.technical_terms) {
              setDefinitions(prevDefs => {
                const prevTerms = new Set(prevDefs.map(d => d.term));
                const newDefs = llmData.llm_output.technical_terms.filter(d => d.term && !prevTerms.has(d.term));
                const merged = [...prevDefs, ...newDefs];
                localStorage.setItem(DEFINITIONS_CACHE_KEY, JSON.stringify(merged));
                return merged;
              });
            }
          }
        } catch (err) {
          console.error('Error fetching LLM output:', err);
        }
      }, 3000);
    } else {
      clearInterval(pollingRef.current);
    }

    return () => clearInterval(pollingRef.current);
  }, [isRecording]);

  const getRecentTranscription = (milliseconds) => {
    const cutoff = new Date(Date.now() - milliseconds);
    return transcriptionBuffer.current
      .filter(item => item.timestamp >= cutoff)
      .map(item => item.text)
      .join(' ');
  };

  const handleStart = async () => {
    if (!isSupported) {
      alert('Speech recognition is not supported in this browser. Please use Chrome, Edge, or Safari.');
      return;
    }

    setTranscription('');
    setInterimTranscription('');
    setLoading(true);
    setHasTranscribed(false);
    transcriptionBuffer.current = [];
    lastSentTime.current = 0;

    try {
      // Start session on server
      const res = await fetch(`${API_URL}/session/start`, { method: 'POST' });
      if (res.ok) {
        recognitionRef.current.start();
      } else {
        setLoading(false);
        alert('Failed to start session on server.');
      }
    } catch (err) {
      setLoading(false);
      alert('Could not start session: ' + err.message);
    }
  };

  const handleStop = async () => {
    if (recognitionRef.current) {
      recognitionRef.current.stop();
    }
    
    setLoading(true);
    try {
      const res = await fetch(`${API_URL}/session/stop`, { method: 'POST' });
      setLoading(false);
      if (!res.ok) {
        alert('Failed to stop session on server.');
      }
    } catch (err) {
      setLoading(false);
      alert('Could not stop session: ' + err.message);
    }
  };

  const handleClear = () => {
    setDefinitions([]);
    localStorage.removeItem(DEFINITIONS_CACHE_KEY);
    setTranscription('');
    setInterimTranscription('');
    setHasTranscribed(false);
    transcriptionBuffer.current = [];
  };

  const DefinitionCard = ({ term, definition }) => (
    <div className="card definition-card">
      <div className="card-term">{term || <i>Unknown term</i>}</div>
      <div className="card-definition">{definition || <i>No definition</i>}</div>
    </div>
  );

  const ContextCard = ({ term, contextual_explanation, example_quote }) => (
    <div className="card context-card">
      <div className="card-term">{term || <i>Unknown term</i>}</div>
      <div className="card-context">{contextual_explanation || <i>No context</i>}</div>
      {example_quote && <div className="card-example"><b>Example:</b> "{example_quote}"</div>}
    </div>
  );

  const renderDefinitions = () => {
    if (!definitions || definitions.length === 0) {
      return <div>No technical terms found yet.</div>;
    }
    return (
      <div className="card-list">
        {definitions.map((def, idx) => (
          <DefinitionCard key={idx} term={def.term} definition={def.definition} />
        ))}
      </div>
    );
  };

  const renderContext = () => {
    if (!definitions || definitions.length === 0) {
      return <div>No contextual explanations yet.</div>;
    }
    return (
      <div className="card-list">
        {definitions.map((def, idx) => (
          <ContextCard key={idx} term={def.term} contextual_explanation={def.contextual_explanation} example_quote={def.example_quote} />
        ))}
      </div>
    );
  };

  if (!isSupported) {
    return (
      <div className={`app-root ${darkMode ? 'dark' : 'light'}`}>
        <div style={{ padding: '20px', textAlign: 'center' }}>
          <h2>Browser Not Supported</h2>
          <p>Web Speech API is not supported in this browser.</p>
          <p>Please use Chrome, Edge, or Safari for speech recognition.</p>
        </div>
      </div>
    );
  }

  return (
    <div className={`app-root ${darkMode ? 'dark' : 'light'}`}>
      <header className="app-header glass">
        <div className="header-title">Meeting Insights</div>
        <div className="header-actions">
          <button className="icon-btn" title="GitHub Repo">
            <FaGithub />
          </button>
          <button className="theme-toggle-pill" onClick={() => setDarkMode(dm => !dm)}>
            {darkMode ? <FaSun /> : <FaMoon />}
          </button>
        </div>
      </header>
      
      <div className="main-columns">
        <div className="column transcript-column glass">
          <div className="transcript-header">
            <button
              className={`record-btn${isRecording ? ' recording' : ''}`}
              onClick={isRecording ? handleStop : handleStart}
              disabled={loading}
            >
              {isRecording ? <><FaStop /> Stop</> : <><FaMicrophone /> Record</>}
            </button>
            <span className="status-text">
              {isRecording ? 'Recording...' : loading ? 'Processing...' : 'Ready'}
            </span>
            <button
              className="clear-session-btn"
              onClick={handleClear}
              title="Clear all definitions, explanations, and transcript"
            >
              Clear Session
            </button>
          </div>
          <div className="transcription-box glass">
            <div className="transcription-content">
              {!hasTranscribed && (
                <div className="greeting"></div>
              )}
              <div className="mic-visualizer">
                {isRecording ? (
                  <div className="mic-anim">
                    <FaMicrophone className="mic-icon recording" />
                  </div>
                ) : (
                  <FaMicrophone className="mic-icon idle" />
                )}
              </div>
              <div className="transcription-text fade-in">
                {loading ? (
                  <span>Starting...</span>
                ) : (
                  <span>
                    {transcription || 'Hi there! Start speaking to transcribe your voice note.'}
                    <span style={{ color: '#999' }}>{interimTranscription}</span>
                  </span>
                )}
              </div>
            </div>
          </div>
        </div>
        
        <div className="column definitions-column glass">
          <div className="column-title">Technical Definitions</div>
          <div className="definitions-box card-scroll">
            {renderDefinitions()}
          </div>
        </div>
        
        <div className="column context-column glass">
          <div className="column-title">Contextual Explanations & Examples</div>
          <div className="context-box card-scroll">
            {renderContext()}
          </div>
        </div>
      </div>
    </div>
  );
}

export default App;
```

And here's your updated FastAPI server (simplified since transcription is now client-side):

```python
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

# Load environment variables
load_dotenv()
GEMINI_API_KEY = os.getenv('GOOGLE_API_KEY')
LLM_OUTPUT_FILE = "llm_definitions.jsonl"

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

# Shared data
class SharedState:
    def __init__(self):
        self.transcription_buffer = []  # list of (timestamp, text)
        self.llm_output = None
        self.lock = threading.Lock()

    def add_transcription(self, text):
        with self.lock:
            self.transcription_buffer.append((datetime.utcnow(), text))
            # Keep only last 30 seconds
            cutoff = datetime.utcnow() - timedelta(seconds=30)
            self.transcription_buffer = [x for x in self.transcription_buffer if x[0] >= cutoff]

    def get_last_n_seconds(self, seconds=5):
        with self.lock:
            cutoff = datetime.utcnow() - timedelta(seconds=seconds)
            return ' '.join([t for ts, t in self.transcription_buffer if ts >= cutoff])

    def set_llm_output(self, output):
        with self.lock:
            self.llm_output = output

    def get_llm_output(self):
        with self.lock:
            return self.llm_output

shared_state = SharedState()

# Gemini LLM call (same as before)
SCHEMA = '''
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

def get_gemini_definitions(text):
    prompt = (
        "Extract all technical terms from the following text and provide their definitions and context. "
        "Return the result as a JSON object matching this schema (if no terms, use an empty list for 'technical_terms'):\n" + SCHEMA + "\nText:\n" + text
    )
    model = genai.GenerativeModel('gemini-2.0-flash')
    response = model.generate_content(prompt)
    match = re.search(r'\{[\s\S]*\}', response.text)
    if match:
        json_str = match.group(0)
        try:
            return json.loads(json_str)
        except Exception as e:
            print(f"[LLM JSON ERROR] {e}\nRaw output: {json_str}")
            return None
    else:
        print(f"[LLM NO JSON FOUND] Raw output: {response.text}")
        return None

# Background LLM processor
class LLMWorker(threading.Thread):
    def __init__(self, shared_state, interval=5):
        super().__init__(daemon=True)
        self.shared_state = shared_state
        self.interval = interval
        self.running = llm_running

    def run(self):
        while self.running.is_set():
            time.sleep(self.interval)
            recent_text = self.shared_state.get_last_n_seconds(self.interval)
            if recent_text.strip():
                definitions = get_gemini_definitions(recent_text)
                if definitions:
                    obj = {
                        "timestamp": datetime.utcnow().isoformat(),
                        "transcript": recent_text,
                        "llm_output": definitions
                    }
                    with open(LLM_OUTPUT_FILE, 'a', encoding='utf-8') as f:
                        f.write(json.dumps(obj) + '\n')
                    self.shared_state.set_llm_output(obj)

# API Models
class StatusResponse(BaseModel):
    running: bool

class TranscriptionRequest(BaseModel):
    text: str

@app.post("/session/start", response_model=StatusResponse)
def start_session():
    global llm_thread
    if llm_running.is_set():
        return {"running": True}
    
    llm_running.set()
    shared_state.transcription_buffer.clear()
    shared_state.llm_output = None
    
    llm_thread = LLMWorker(shared_state)
    llm_thread.start()
    
    return {"running": True}

@app.post("/session/stop", response_model=StatusResponse)
def stop_session():
    llm_running.clear()
    return {"running": False}

@app.post("/process-transcription")
def process_transcription(request: TranscriptionRequest):
    """Receive transcription from client and add to buffer for LLM processing"""
    shared_state.add_transcription(request.text)
    return {"status": "received"}

@app.get("/llm/latest")
def get_latest_llm():
    output = shared_state.get_llm_output()
    if output is None:
        raise HTTPException(status_code=404, detail="No LLM output yet.")
    return output

# Health check
@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

## Key Changes:

### React App:
1. **Removed server-side transcription calls** - No more `/transcription/start`, `/transcription/stop`, or `/transcription/live`
2. **Added Web Speech API integration** - Client-side speech recognition with interim results
3. **Added browser support detection** - Graceful fallback for unsupported browsers
4. **New transcription buffer** - Tracks recent transcriptions to send to server
5. **Added `/process-transcription` endpoint calls** - Sends recent speech to server for LLM processing

### FastAPI Server:
1. **Removed Whisper and speech_recognition dependencies** - No longer needed
2. **Simplified to LLM processing only** - Just handles technical term extraction
3. **New `/process-transcription` endpoint** - Receives text from client
4. **Session management** - `/session/start` and `/session/stop` for LLM processing
5. **Removed audio processing** - No more microphone handling

## Benefits:
- **Faster response times** - No audio upload/processing delays
- **Better real-time experience** - Immediate transcription feedback
- **Reduced server load** - No audio processing on server
- **Better privacy** - Audio stays on client device
- **Simpler deployment** - No need for audio libraries on server

## Browser Requirements:
- Chrome, Edge, Safari (best support)
- HTTPS required for production
- Microphone permissions needed

The app will now transcribe speech locally using the Web Speech API while still sending the text to your server for technical term extraction using Gemini AI!
