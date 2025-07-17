Here are the code modifications to provide a real-time feel configuration:

## 1. Updated React App with Real-time Config

```jsx
import React, { useState, useEffect, useRef } from 'react';
import './App.css';
import { FaMicrophone, FaGithub, FaSun, FaMoon, FaStop } from 'react-icons/fa';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:8000';
const DEFINITIONS_CACHE_KEY = 'wf_teams_definitions_cache';

// Real-time configuration
const REALTIME_CONFIG = {
  CLIENT_POLLING_INTERVAL: parseInt(process.env.REACT_APP_CLIENT_POLLING_INTERVAL) || 1000, // 1 second
  LLM_SEND_INTERVAL: parseInt(process.env.REACT_APP_LLM_SEND_INTERVAL) || 2000, // 2 seconds
  TRANSCRIPTION_WINDOW: parseInt(process.env.REACT_APP_TRANSCRIPTION_WINDOW) || 8000, // 8 seconds
  ACTIVITY_BOOST_MULTIPLIER: parseFloat(process.env.REACT_APP_ACTIVITY_BOOST) || 0.5, // 50% faster when active
  QUIET_PERIOD_THRESHOLD: parseInt(process.env.REACT_APP_QUIET_THRESHOLD) || 3000, // 3 seconds of quiet
};

function App() {
  const [isRecording, setIsRecording] = useState(false);
  const [transcription, setTranscription] = useState('');
  const [interimTranscription, setInterimTranscription] = useState('');
  const [loading, setLoading] = useState(false);
  const [darkMode, setDarkMode] = useState(false);
  const [hasTranscribed, setHasTranscribed] = useState(false);
  const [definitions, setDefinitions] = useState([]);
  const [isSupported, setIsSupported] = useState(false);
  const [lastActivityTime, setLastActivityTime] = useState(0);
  
  const recognitionRef = useRef(null);
  const pollingTimeoutRef = useRef(null);
  const transcriptionBuffer = useRef([]);
  const lastSentTime = useRef(0);
  const isActiveRef = useRef(false);

  // Dynamic interval calculation based on activity
  const getPollingInterval = () => {
    const now = Date.now();
    const timeSinceLastActivity = now - lastActivityTime;
    const isQuiet = timeSinceLastActivity > REALTIME_CONFIG.QUIET_PERIOD_THRESHOLD;
    
    if (isQuiet) {
      return REALTIME_CONFIG.CLIENT_POLLING_INTERVAL * 2; // Slower when quiet
    } else {
      return REALTIME_CONFIG.CLIENT_POLLING_INTERVAL * REALTIME_CONFIG.ACTIVITY_BOOST_MULTIPLIER; // Faster when active
    }
  };

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
        const now = Date.now();

        for (let i = event.resultIndex; i < event.results.length; i++) {
          const transcript = event.results[i][0].transcript;
          if (event.results[i].isFinal) {
            finalTranscript += transcript;
          } else {
            interimTranscript += transcript;
          }
        }

        // Update activity time on any speech input
        if (finalTranscript || interimTranscript) {
          setLastActivityTime(now);
          isActiveRef.current = true;
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
        setLastActivityTime(Date.now());
      };

      recognition.onend = () => {
        setIsRecording(false);
        setInterimTranscription('');
        isActiveRef.current = false;
      };

      recognition.onerror = (event) => {
        console.error('Speech recognition error:', event.error);
        setIsRecording(false);
        setLoading(false);
        isActiveRef.current = false;
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
  }, [lastActivityTime]);

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

  // Adaptive polling with dynamic intervals
  useEffect(() => {
    if (isRecording) {
      const schedulePoll = () => {
        pollingTimeoutRef.current = setTimeout(async () => {
          // Send recent transcription to server for LLM processing
          const now = Date.now();
          const recentTranscription = getRecentTranscription(REALTIME_CONFIG.TRANSCRIPTION_WINDOW);
          
          if (recentTranscription && now - lastSentTime.current > REALTIME_CONFIG.LLM_SEND_INTERVAL) {
            try {
              await fetch(`${API_URL}/process-transcription`, {
                method: 'POST',
                headers: {
                  'Content-Type': 'application/json',
                },
                body: JSON.stringify({ 
                  text: recentTranscription,
                  timestamp: now,
                  is_active: isActiveRef.current
                })
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

          // Schedule next poll with dynamic interval
          if (isRecording) {
            schedulePoll();
          }
        }, getPollingInterval());
      };

      schedulePoll();
    } else {
      clearTimeout(pollingTimeoutRef.current);
    }

    return () => clearTimeout(pollingTimeoutRef.current);
  }, [isRecording, lastActivityTime]);

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
    setLastActivityTime(Date.now());
    isActiveRef.current = false;

    try {
      // Start session on server with real-time config
      const res = await fetch(`${API_URL}/session/start`, { 
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          config: {
            llm_interval: Math.floor(REALTIME_CONFIG.LLM_SEND_INTERVAL / 1000), // Convert to seconds
            transcription_window: Math.floor(REALTIME_CONFIG.TRANSCRIPTION_WINDOW / 1000),
            realtime_mode: true
          }
        })
      });
      
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
    
    clearTimeout(pollingTimeoutRef.current);
    setLoading(true);
    isActiveRef.current = false;
    
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
    setLastActivityTime(0);
    isActiveRef.current = false;
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
        <div className="header-subtitle">
          Real-time Mode • {getPollingInterval()}ms polling
        </div>
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
              {isRecording ? 
                `Recording... ${isActiveRef.current ? '🎤' : '⏸️'}` : 
                loading ? 'Processing...' : 'Ready'
              }
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
                    <FaMicrophone className={`mic-icon recording ${isActiveRef.current ? 'active' : 'listening'}`} />
                  </div>
                ) : (
                  <FaMicrophone className="mic-icon idle" />
                )}
              </div>
              <div className="transcription-text fade-in">
                {loading ? (
                  <span>Starting real-time mode...</span>
                ) : (
                  <span>
                    {transcription || 'Hi there! Start speaking to transcribe your voice note.'}
                    <span style={{ color: '#999', fontStyle: 'italic' }}>{interimTranscription}</span>
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

## 2. Updated FastAPI Server with Real-time Config

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
from typing import Optional, Dict, Any

# Load environment variables
load_dotenv()
GEMINI_API_KEY = os.getenv('GOOGLE_API_KEY')
LLM_OUTPUT_FILE = "llm_definitions.jsonl"

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

    def update_config(self, new_config):
        with self.lock:
            self.config.update(new_config)

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
        print(f"[LLM NO JSON FOUND] Raw output: {response.te
