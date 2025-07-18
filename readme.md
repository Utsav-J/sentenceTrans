import React, { useState, useEffect, useRef } from 'react';
import './App.css';
import { FaMicrophone, FaGithub, FaSun, FaMoon, FaStop, FaClipboardList, FaExclamationCircle, FaCheckCircle, FaStickyNote } from 'react-icons/fa';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:8000';
const DEFINITIONS_CACHE_KEY = 'wf_teams_definitions_cache';
const ACTION_ITEMS_CACHE_KEY = 'wf_teams_action_items_cache';

function App() {
  const [isRecording, setIsRecording] = useState(false);
  const [transcription, setTranscription] = useState('');
  const [interimTranscription, setInterimTranscription] = useState('');
  const [loading, setLoading] = useState(false);
  const [darkMode, setDarkMode] = useState(false);
  const [hasTranscribed, setHasTranscribed] = useState(false);
  const [definitions, setDefinitions] = useState([]);
  const [actionItems, setActionItems] = useState([]);
  const [isSupported, setIsSupported] = useState(false);
  const [activeTab, setActiveTab] = useState('definitions'); // 'definitions', 'context', 'actions'
  
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

  // Load cached data on mount
  useEffect(() => {
    const cachedDefinitions = localStorage.getItem(DEFINITIONS_CACHE_KEY);
    if (cachedDefinitions) {
      try {
        setDefinitions(JSON.parse(cachedDefinitions));
      } catch {
        setDefinitions([]);
      }
    }

    const cachedActionItems = localStorage.getItem(ACTION_ITEMS_CACHE_KEY);
    if (cachedActionItems) {
      try {
        setActionItems(JSON.parse(cachedActionItems));
      } catch {
        setActionItems([]);
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

        // Get latest LLM output (technical definitions)
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

        // Get latest action items
        try {
          const actionRes = await fetch(`${API_URL}/action-items/latest`);
          if (actionRes.ok) {
            const actionData = await actionRes.json();
            if (actionData.action_items && actionData.action_items.length > 0) {
              setActionItems(prevItems => {
                const allItems = [];
                actionData.action_items.forEach(sessionData => {
                  if (sessionData.action_items && sessionData.action_items.action_items) {
                    sessionData.action_items.action_items.forEach(item => {
                      allItems.push({
                        ...item,
                        timestamp: sessionData.timestamp,
                        id: `${sessionData.timestamp}-${item.title}` // Simple ID generation
                      });
                    });
                  }
                });
                
                // Merge with existing items, avoiding duplicates
                const existingIds = new Set(prevItems.map(item => item.id));
                const newItems = allItems.filter(item => !existingIds.has(item.id));
                const merged = [...prevItems, ...newItems];
                localStorage.setItem(ACTION_ITEMS_CACHE_KEY, JSON.stringify(merged));
                return merged;
              });
            }
          }
        } catch (err) {
          console.error('Error fetching action items:', err);
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
    setActionItems([]);
    localStorage.removeItem(DEFINITIONS_CACHE_KEY);
    localStorage.removeItem(ACTION_ITEMS_CACHE_KEY);
    setTranscription('');
    setInterimTranscription('');
    setHasTranscribed(false);
    transcriptionBuffer.current = [];
  };

  const getPriorityIcon = (priority) => {
    switch (priority?.toLowerCase()) {
      case 'high':
        return <FaExclamationCircle className="priority-icon high" />;
      case 'medium':
        return <FaExclamationCircle className="priority-icon medium" />;
      case 'low':
        return <FaCheckCircle className="priority-icon low" />;
      default:
        return <FaStickyNote className="priority-icon default" />;
    }
  };

  const getActionTypeIcon = (type) => {
    switch (type?.toLowerCase()) {
      case 'note':
        return <FaStickyNote className="action-type-icon note" />;
      case 'form':
        return <FaClipboardList className="action-type-icon form" />;
      case 'task':
        return <FaCheckCircle className="action-type-icon task" />;
      case 'reminder':
        return <FaExclamationCircle className="action-type-icon reminder" />;
      default:
        return <FaClipboardList className="action-type-icon default" />;
    }
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

  const ActionItemCard = ({ title, description, type, priority, context, suggested_content, timestamp }) => (
    <div className="card action-item-card">
      <div className="action-item-header">
        <div className="action-item-title">
          {getActionTypeIcon(type)}
          <span>{title || <i>Untitled Action</i>}</span>
        </div>
        <div className="action-item-priority">
          {getPriorityIcon(priority)}
          <span className={`priority-text ${priority?.toLowerCase()}`}>
            {priority || 'Normal'}
          </span>
        </div>
      </div>
      <div className="action-item-description">
        {description || <i>No description</i>}
      </div>
      {suggested_content && (
        <div className="action-item-content">
          <b>Suggested Content:</b> {suggested_content}
        </div>
      )}
      <div className="action-item-context">
        <b>Context:</b> {context || <i>No context</i>}
      </div>
      <div className="action-item-meta">
        <span className="action-type">{type || 'Action'}</span>
        {timestamp && (
          <span className="action-timestamp">
            {new Date(timestamp).toLocaleTimeString()}
          </span>
        )}
      </div>
    </div>
  );

  const renderDefinitions = () => {
    if (!definitions || definitions.length === 0) {
      return <div className="empty-state">No technical terms found yet.</div>;
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
      return <div className="empty-state">No contextual explanations yet.</div>;
    }
    return (
      <div className="card-list">
        {definitions.map((def, idx) => (
          <ContextCard key={idx} term={def.term} contextual_explanation={def.contextual_explanation} example_quote={def.example_quote} />
        ))}
      </div>
    );
  };

  const renderActionItems = () => {
    if (!actionItems || actionItems.length === 0) {
      return <div className="empty-state">No action items detected yet.</div>;
    }
    return (
      <div className="card-list">
        {actionItems.map((item, idx) => (
          <ActionItemCard 
            key={item.id || idx} 
            title={item.title}
            description={item.description}
            type={item.type}
            priority={item.priority}
            context={item.context}
            suggested_content={item.suggested_content}
            timestamp={item.timestamp}
          />
        ))}
      </div>
    );
  };

  const getTabContent = () => {
    switch (activeTab) {
      case 'definitions':
        return renderDefinitions();
      case 'context':
        return renderContext();
      case 'actions':
        return renderActionItems();
      default:
        return renderDefinitions();
    }
  };

  const getTabTitle = () => {
    switch (activeTab) {
      case 'definitions':
        return 'Technical Definitions';
      case 'context':
        return 'Contextual Explanations';
      case 'actions':
        return 'Action Items';
      default:
        return 'Technical Definitions';
    }
  };

  const getTabCounts = () => {
    return {
      definitions: definitions.length,
      context: definitions.length,
      actions: actionItems.length
    };
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

  const tabCounts = getTabCounts();

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
      
      <div className="main-layout">
        <div className="left-column transcript-column glass">
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
              title="Clear all definitions, explanations, action items, and transcript"
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
        
        <div className="right-column glass">
          <div className="tab-navigation">
            <button
              className={`tab-btn ${activeTab === 'definitions' ? 'active' : ''}`}
              onClick={() => setActiveTab('definitions')}
            >
              <FaClipboardList />
              Definitions
              {tabCounts.definitions > 0 && <span className="tab-count">{tabCounts.definitions}</span>}
            </button>
            <button
              className={`tab-btn ${activeTab === 'context' ? 'active' : ''}`}
              onClick={() => setActiveTab('context')}
            >
              <FaStickyNote />
              Context
              {tabCounts.context > 0 && <span className="tab-count">{tabCounts.context}</span>}
            </button>
            <button
              className={`tab-btn ${activeTab === 'actions' ? 'active' : ''}`}
              onClick={() => setActiveTab('actions')}
            >
              <FaExclamationCircle />
              Actions
              {tabCounts.actions > 0 && <span className="tab-count">{tabCounts.actions}</span>}
            </button>
          </div>
          
          <div className="tab-content">
            <div className="column-title">{getTabTitle()}</div>
            <div className="content-box card-scroll">
              {getTabContent()}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

export default App;
