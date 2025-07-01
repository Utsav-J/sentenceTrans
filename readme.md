Here's the code to display a top-right corner banner for 3 seconds when the Chainlit agent is initialized:

**Python code (for your `on_chat_start` function):**

```python
import chainlit as cl

@cl.on_chat_start
async def on_chat_start():
    # Your existing initialization code here...
    
    # Display the initialization banner
    banner_element = cl.CustomElement(
        name="InitializationBanner", 
        props={
            "message": "Agent Initialized Successfully!",
            "duration": 3000  # 3 seconds in milliseconds
        }
    )
    
    await cl.Message(content="", elements=[banner_element]).send()
    
    # Rest of your initialization code...
```

**Frontend JSX component (`public/elements/InitializationBanner.jsx`):**

```jsx
import { useEffect, useState } from 'react'
import { CheckCircle } from 'lucide-react'

export default function InitializationBanner() {
  const [isVisible, setIsVisible] = useState(true)

  useEffect(() => {
    const timer = setTimeout(() => {
      setIsVisible(false)
    }, props.duration || 3000)

    return () => clearTimeout(timer)
  }, [])

  if (!isVisible) return null

  return (
    <div className="fixed top-4 right-4 z-50 animate-in slide-in-from-top-2 duration-300">
      <div className="bg-green-500 text-white px-4 py-3 rounded-lg shadow-lg flex items-center gap-2 min-w-[250px]">
        <CheckCircle className="h-5 w-5" />
        <span className="font-medium">
          {props.message || "Agent Initialized!"}
        </span>
      </div>
    </div>
  )
}
```

**Alternative with fade-out animation (`public/elements/InitializationBanner.jsx`):**

```jsx
import { useEffect, useState } from 'react'
import { CheckCircle } from 'lucide-react'

export default function InitializationBanner() {
  const [isVisible, setIsVisible] = useState(true)
  const [shouldRender, setShouldRender] = useState(true)

  useEffect(() => {
    const hideTimer = setTimeout(() => {
      setIsVisible(false)
    }, (props.duration || 3000) - 300) // Start fade 300ms before removal

    const removeTimer = setTimeout(() => {
      setShouldRender(false)
    }, props.duration || 3000)

    return () => {
      clearTimeout(hideTimer)
      clearTimeout(removeTimer)
    }
  }, [])

  if (!shouldRender) return null

  return (
    <div 
      className={`fixed top-4 right-4 z-50 transition-all duration-300 ${
        isVisible 
          ? 'opacity-100 translate-y-0' 
          : 'opacity-0 -translate-y-2'
      }`}
    >
      <div className="bg-green-500 text-white px-4 py-3 rounded-lg shadow-lg flex items-center gap-2 min-w-[250px]">
        <CheckCircle className="h-5 w-5" />
        <span className="font-medium">
          {props.message || "Agent Initialized!"}
        </span>
      </div>
    </div>
  )
}
```

**Usage in your `on_chat_start`:**

```python
@cl.on_chat_start
async def on_chat_start():
    # Show initialization banner
    banner = cl.CustomElement(
        name="InitializationBanner", 
        props={
            "message": "🤖 AI Agent Ready!",
            "duration": 3000
        }
    )
    await cl.Message(content="", elements=[banner]).send()
    
    # Your agent initialization code
    # ... rest of your setup
```

The banner will appear in the top-right corner with a success checkmark, display for 3 seconds, then automatically disappear. You can customize the message and duration through the props.
