To implement context/memory reset logic based on whether a question is a follow-up or not, you'll need to detect the nature of the question and manage your memory accordingly. Here are several approaches:

## Method 1: Using Keywords and Context Clues

```python
import chainlit as cl
from datetime import datetime, timedelta

# Store conversation state
conversation_memory = []
last_message_time = None

def is_followup_question(message: str, previous_context: list) -> bool:
    """Determine if the message is a follow-up question"""
    
    # Keywords that typically indicate follow-ups
    followup_indicators = [
        'also', 'additionally', 'furthermore', 'moreover', 'and',
        'what about', 'how about', 'can you also', 'tell me more',
        'explain further', 'elaborate', 'continue', 'go on',
        'that', 'this', 'it', 'they', 'them'  # pronouns referring to previous context
    ]
    
    # Check for pronouns and references
    pronoun_references = ['it', 'this', 'that', 'these', 'those', 'they', 'them']
    
    message_lower = message.lower()
    
    # Check for explicit follow-up indicators
    for indicator in followup_indicators:
        if indicator in message_lower:
            return True
    
    # Check for pronouns that likely refer to previous context
    if any(pronoun in message_lower.split()[:5] for pronoun in pronoun_references):
        return True
    
    # Check if message is very short (likely a follow-up)
    if len(message.split()) <= 3 and previous_context:
        return True
    
    return False

def is_time_based_continuation() -> bool:
    """Check if message came within a short time window"""
    global last_message_time
    
    if last_message_time is None:
        return False
    
    time_diff = datetime.now() - last_message_time
    return time_diff < timedelta(minutes=2)  # Adjust threshold as needed

@cl.on_message
async def main(message: cl.Message):
    global conversation_memory, last_message_time
    
    user_message = message.content
    
    # Determine if this is a follow-up
    is_followup = (
        is_followup_question(user_message, conversation_memory) or 
        is_time_based_continuation()
    )
    
    if not is_followup:
        # Reset memory for new conversation
        conversation_memory = []
        await cl.Message(content="🔄 Starting new conversation context...").send()
    
    # Add current message to memory
    conversation_memory.append({
        "role": "user",
        "content": user_message,
        "timestamp": datetime.now()
    })
    
    # Generate response using memory
    response = generate_response(user_message, conversation_memory if is_followup else [])
    
    # Add response to memory
    conversation_memory.append({
        "role": "assistant", 
        "content": response,
        "timestamp": datetime.now()
    })
    
    # Update timestamp
    last_message_time = datetime.now()
    
    # Send response
    await cl.Message(content=response).send()
```

## Method 2: Using User Session State

```python
import chainlit as cl

@cl.on_chat_start
async def start():
    # Initialize session state
    cl.user_session.set("conversation_memory", [])
    cl.user_session.set("last_topic", None)
    cl.user_session.set("message_count", 0)

def detect_topic_change(current_message: str, last_topic: str) -> bool:
    """Simple topic change detection"""
    if not last_topic:
        return True
    
    # You can implement more sophisticated topic modeling here
    # For now, using simple keyword overlap
    current_words = set(current_message.lower().split())
    last_words = set(last_topic.lower().split())
    
    # If less than 20% word overlap, consider it a topic change
    if len(current_words & last_words) / max(len(current_words), 1) < 0.2:
        return True
    
    return False

@cl.on_message
async def main(message: cl.Message):
    user_message = message.content
    
    # Get session state
    memory = cl.user_session.get("conversation_memory", [])
    last_topic = cl.user_session.get("last_topic")
    message_count = cl.user_session.get("message_count", 0)
    
    # Determine if we should reset context
    should_reset = (
        message_count == 0 or  # First message
        detect_topic_change(user_message, last_topic) or
        len(memory) > 10  # Reset after too many exchanges
    )
    
    if should_reset:
        memory = []
        await cl.Message(content="🆕 New conversation started").send()
    
    # Add to memory
    memory.append({"role": "user", "content": user_message})
    
    # Generate response
    response = generate_response_with_context(user_message, memory)
    memory.append({"role": "assistant", "content": response})
    
    # Update session state
    cl.user_session.set("conversation_memory", memory)
    cl.user_session.set("last_topic", user_message)
    cl.user_session.set("message_count", message_count + 1)
    
    await cl.Message(content=response).send()
```

## Method 3: Explicit User Intent Detection

```python
import re
import chainlit as cl

def analyze_user_intent(message: str, has_context: bool) -> dict:
    """Analyze user message to determine intent and context needs"""
    
    message_lower = message.lower().strip()
    
    # Explicit reset triggers
    reset_triggers = [
        'new topic', 'change subject', 'start over', 'reset',
        'different question', 'something else', 'new question'
    ]
    
    # Follow-up indicators
    followup_indicators = [
        'also', 'and', 'but', 'however', 'what about',
        'can you also', 'tell me more', 'explain',
        'elaborate on', 'continue', 'more details'
    ]
    
    # Question starters that usually indicate new topics
    new_topic_starters = [
        'how do i', 'what is', 'can you help', 'i need to',
        'i want to', 'how can i', 'what are', 'tell me about'
    ]
    
    intent = {
        'is_followup': False,
        'is_new_topic': False,
        'confidence': 0.0
    }
    
    # Check for explicit reset
    if any(trigger in message_lower for trigger in reset_triggers):
        intent['is_new_topic'] = True
        intent['confidence'] = 0.9
        return intent
    
    # Check for follow-up indicators
    if any(indicator in message_lower for indicator in followup_indicators):
        intent['is_followup'] = True
        intent['confidence'] = 0.8
        return intent
    
    # Check for new topic starters
    if any(message_lower.startswith(starter) for starter in new_topic_starters):
        intent['is_new_topic'] = True
        intent['confidence'] = 0.7
        return intent
    
    # If no context exists, treat as new topic
    if not has_context:
        intent['is_new_topic'] = True
        intent['confidence'] = 1.0
        return intent
    
    # Default: assume follow-up if context exists
    intent['is_followup'] = has_context
    intent['confidence'] = 0.5
    
    return intent

@cl.on_message
async def main(message: cl.Message):
    user_message = message.content
    
    # Get current memory
    memory = cl.user_session.get("conversation_memory", [])
    
    # Analyze intent
    intent = analyze_user_intent(user_message, len(memory) > 0)
    
    # Decision logic
    if intent['is_new_topic'] or (not intent['is_followup'] and intent['confidence'] > 0.6):
        # Reset memory
        memory = []
        context_msg = f"🔄 New conversation (confidence: {intent['confidence']:.1f})"
        await cl.Message(content=context_msg).send()
    
    # Process with appropriate context
    memory.append({"role": "user", "content": user_message})
    
    # Your response generation logic here
    response = f"Processing with {'existing' if len(memory) > 1 else 'new'} context..."
    
    memory.append({"role": "assistant", "content": response})
    cl.user_session.set("conversation_memory", memory)
    
    await cl.Message(content=response).send()
```

## Method 4: Simple Time + Content Based Approach

```python
import chainlit as cl
from datetime import datetime, timedelta

@cl.on_chat_start
async def start():
    cl.user_session.set("last_message_time", None)
    cl.user_session.set("conversation_memory", [])

@cl.on_message
async def main(message: cl.Message):
    current_time = datetime.now()
    last_time = cl.user_session.get("last_message_time")
    memory = cl.user_session.get("conversation_memory", [])
    
    # Reset conditions
    should_reset = (
        last_time is None or  # First message
        (current_time - last_time) > timedelta(minutes=5) or  # Long gap
        len(message.content.split()) > 20  # Long message (likely new topic)
    )
    
    if should_reset:
        memory = []
        if last_time is not None:
            await cl.Message(content="⏰ Context reset due to time gap or new topic").send()
    
    # Continue with your logic...
    memory.append({"role": "user", "content": message.content})
    
    # Update session
    cl.user_session.set("last_message_time", current_time)
    cl.user_session.set("conversation_memory", memory)
    
    # Generate and send response
    response = generate_response(message.content, memory)
    await cl.Message(content=response).send()
```

Choose the method that best fits your application's needs. Method 1 provides the most sophisticated detection, while Method 4 is simpler and more reliable for basic use cases.
