/* Enhanced App.css with Action Items Support */

:root {
  --primary-color: #4f46e5;
  --secondary-color: #7c3aed;
  --accent-color: #06b6d4;
  --success-color: #10b981;
  --warning-color: #f59e0b;
  --error-color: #ef4444;
  --bg-primary: #ffffff;
  --bg-secondary: #f8fafc;
  --bg-tertiary: #f1f5f9;
  --text-primary: #1e293b;
  --text-secondary: #64748b;
  --text-muted: #94a3b8;
  --border-color: #e2e8f0;
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
  --glass-bg: rgba(255, 255, 255, 0.8);
  --glass-border: rgba(255, 255, 255, 0.2);
  --glass-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
}

[data-theme="dark"] {
  --bg-primary: #0f172a;
  --bg-secondary: #1e293b;
  --bg-tertiary: #334155;
  --text-primary: #f8fafc;
  --text-secondary: #cbd5e1;
  --text-muted: #64748b;
  --border-color: #334155;
  --glass-bg: rgba(15, 23, 42, 0.8);
  --glass-border: rgba(255, 255, 255, 0.1);
  --glass-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.37);
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen',
    'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  background: linear-gradient(135deg, var(--bg-primary) 0%, var(--bg-secondary) 100%);
  color: var(--text-primary);
  min-height: 100vh;
  transition: all 0.3s ease;
}

.app-root {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.glass {
  background: var(--glass-bg);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border: 1px solid var(--glass-border);
  box-shadow: var(--glass-shadow);
}

/* Header */
.app-header {
  padding: 1rem 2rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-radius: 0;
  margin-bottom: 1rem;
  position: sticky;
  top: 0;
  z-index: 100;
}

.header-title {
  font-size: 1.5rem;
  font-weight: 700;
  color: var(--primary-color);
}

.header-actions {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.icon-btn {
  padding: 0.5rem;
  border: none;
  background: rgba(79, 70, 229, 0.1);
  color: var(--primary-color);
  border-radius: 0.5rem;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

.icon-btn:hover {
  background: rgba(79, 70, 229, 0.2);
  transform: translateY(-2px);
}

.theme-toggle-pill {
  padding: 0.5rem 1rem;
  border: none;
  background: var(--primary-color);
  color: white;
  border-radius: 2rem;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.theme-toggle-pill:hover {
  background: var(--secondary-color);
  transform: translateY(-2px);
}

/* Main Layout */
.main-layout {
  flex: 1;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
  padding: 0 2rem 2rem;
  max-width: 1400px;
  margin: 0 auto;
  width: 100%;
}

.left-column {
  display: flex;
  flex-direction: column;
  border-radius: 1rem;
  overflow: hidden;
}

.right-column {
  display: flex;
  flex-direction: column;
  border-radius: 1rem;
  overflow: hidden;
}

/* Transcript Column */
.transcript-column {
  padding: 1.5rem;
}

.transcript-header {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.record-btn {
  padding: 0.75rem 1.5rem;
  border: none;
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: white;
  border-radius: 2rem;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-weight: 600;
  font-size: 0.9rem;
}

.record-btn:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-lg);
}

.record-btn.recording {
  background: linear-gradient(135deg, var(--error-color), #dc2626);
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.8; }
}

.status-text {
  color: var(--text-secondary);
  font-size: 0.9rem;
  font-weight: 500;
}

.clear-session-btn {
  padding: 0.5rem 1rem;
  border: 1px solid var(--border-color);
  background: transparent;
  color: var(--text-secondary);
  border-radius: 0.5rem;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.9rem;
}

.clear-session-btn:hover {
  background: var(--error-color);
  color: white;
  border-color: var(--error-color);
}

/* Transcription Box */
.transcription-box {
  flex: 1;
  padding: 2rem;
  border-radius: 1rem;
  display: flex;
  flex-direction: column;
  min-height: 300px;
}

.transcription-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  gap: 1rem;
}

.mic-visualizer {
  margin-bottom: 1rem;
}

.mic-icon {
  font-size: 3rem;
  color: var(--text-muted);
  transition: all 0.3s ease;
}

.mic-icon.recording {
  color: var(--error-color);
  animation: bounce 1s infinite;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
}

.transcription-text {
  font-size: 1.1rem;
  line-height: 1.6;
  color: var(--text-primary);
  max-width: 100%;
  word-wrap: break-word;
}

.fade-in {
  animation: fadeIn 0.5s ease-in;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

/* Tab Navigation */
.tab-navigation {
  display: flex;
  background: var(--bg-tertiary);
  border-radius: 0.5rem;
  padding: 0.25rem;
  margin: 1rem;
  gap: 0.25rem;
}

.tab-btn {
  flex: 1;
  padding: 0.75rem 1rem;
  border: none;
  background: transparent;
  color: var(--text-secondary);
  border-radius: 0.375rem;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  font-weight: 500;
  font-size: 0.9rem;
  position: relative;
}

.tab-btn.active {
  background: var(--primary-color);
  color: white;
  box-shadow: var(--shadow-sm);
}

.tab-btn:hover:not(.active) {
  background: rgba(79, 70, 229, 0.1);
  color: var(--primary-color);
}

.tab-count {
  background: rgba(255, 255, 255, 0.2);
  color: white;
  padding: 0.125rem 0.375rem;
  border-radius: 0.75rem;
  font-size: 0.75rem;
  font-weight: 600;
  min
