Looking at your current code, I can see several ways to enhance it without requiring 2 tool calls. Here are the key improvements:

## 1. Fix the Logic Error

Your current logic has an issue - you're checking if `new_tool_call_signature == last_tool_call_signature` for context reset, but this should be when they're **different**:

```python
# CURRENT (incorrect):
if new_tool_call_signature and new_tool_call_signature == last_tool_call_signature:
    context_reset = True

# SHOULD BE:
if new_tool_call_signature and new_tool_call_signature != last_tool_call_signature:
    context_reset = True
```

## 2. Enhanced Implementation

Here's the improved version that avoids double tool calls:

```python
@cl.on_message
async def main(message: cl.Message):
    connection_info = cl.user_session.get("connection_info")
    if not connection_info or "agent" not in connection_info:
        await cl.Message(content="❌ Agent not initialized. Please refresh the page.").send()
        return

    markdown_links = []
    response_can_be_visualized = False
    actions = []

    async with cl.Step(name="Thinking", type="run") as step:
        passed_content = classify_intent(message.content)
        try:
            # Get conversation state
            message_history = cl.user_session.get("message_history", [])
            is_followup = cl.user_session.get("is_followup", False)
            last_tool_context = cl.user_session.get("last_tool_context", None)
            
            if not message_history:
                message_history.append({"role": "system", "content": SYSTEM_PROMPT})
            
            agent = connection_info["agent"]
            
            # ENHANCED FOLLOW-UP DETECTION
            if is_followup and last_tool_context:
                # Check if this is likely a follow-up question
                if is_likely_followup_question(passed_content, last_tool_context):
                    # Handle as follow-up without running agent first
                    step.output = "Using previous context for follow-up..."
                    
                    enhanced_messages = [{"role": "system", "content": SYSTEM_PROMPT}]
                    json_str = json.dumps(last_tool_context, indent=2) if isinstance(last_tool_context, dict) else str(last_tool_context)
                    enhanced_messages.append({
                        "role": "system",
                        "content": (
                            "You are answering a follow-up question. Here is the previous data context:\n\n"
                            f"{json_str}\n\nUse this data to answer the user's question."
                        )
                    })
                    enhanced_messages.append({"role": "user", "content": passed_content})
                    
                    # Single agent call with context
                    response = await agent.ainvoke({"messages": enhanced_messages})
                    full_messages = response if isinstance(response, list) else []
                    
                    # Check if tools were actually used (override our heuristic)
                    new_tool_message = next((m for m in full_messages if isinstance(m, ToolMessage)), None)
                    if new_tool_message:
                        # Our heuristic was wrong, this did make a new tool call
                        try:
                            tool_content = json.loads(new_tool_message.content)
                        except Exception:
                            tool_content = new_tool_message.content
                        cl.user_session.set("last_tool_context", tool_content)
                        # Process as new tool call
                        called_tool_name = identify_toolcall(full_messages)
                        await process_tool_response(called_tool_name, full_messages, message_history, 
                                                  markdown_links, actions, response_can_be_visualized)
                    else:
                        # True follow-up, keep existing context
                        step.output = "Follow-up response generated!"
                    
                    # Add to message history
                    message_history.append({"role": "user", "content": passed_content})
                    message_history.append({"role": "assistant", "content": str(response)})
                    cl.user_session.set("message_history", message_history)
                    
                    # Send response and exit
                    await send_final_response(response, markdown_links, passed_content, actions)
                    return
            
            # NORMAL PROCESSING (single agent call)
            message_history.append({"role": "user", "content": passed_content})
            response = await agent.ainvoke({"messages": message_history})
            full_messages = response if isinstance(response, list) else []
            
            # Check for tool usage
            first_tool_message = next((m for m in full_messages if isinstance(m, ToolMessage)), None)
            
            if first_tool_message:
                # Tool was used - store context and enable follow-up
                try:
                    tool_content = json.loads(first_tool_message.content)
                except Exception:
                    tool_content = first_tool_message.content
                
                cl.user_session.set("last_tool_context", tool_content)
                cl.user_session.set("is_followup", True)
                
                # Process tool response
                called_tool_name = identify_toolcall(full_messages)
                await process_tool_response(called_tool_name, full_messages, message_history, 
                                          markdown_links, actions, response_can_be_visualized)
            else:
                # No tool used - disable follow-up mode
                cl.user_session.set("is_followup", False)
                cl.user_session.set("last_tool_context", None)
            
            # Add to message history
            message_history.append({"role": "assistant", "content": str(response)})
            cl.user_session.set("message_history", message_history)
            
        except Exception as e:
            step.output = f"Error: {str(e)}"
            response = f"❌ Sorry, I encountered an error: {str(e)}"
            await cl.Message(content=response).send()
            return

    await send_final_response(response, markdown_links, passed_content, actions)

# HELPER FUNCTIONS

def is_likely_followup_question(content, last_tool_context):
    """Determine if this is likely a follow-up question based on content and context"""
    content_lower = content.lower()
    
    # Follow-up indicators
    followup_patterns = [
        "what about", "also", "furthermore", "additionally", "more details",
        "explain", "why", "how", "what does", "tell me more", "elaborate",
        "can you", "show me", "what is", "what are", "which", "where",
        "compare", "difference", "similar", "related", "another"
    ]
    
    # New topic indicators (likely need new tools)
    new_topic_patterns = [
        "now", "instead", "different", "new", "search for", "find", "get",
        "latest", "current", "recent", "update", "refresh", "other topic"
    ]
    
    # Check for follow-up patterns
    has_followup_pattern = any(pattern in content_lower for pattern in followup_patterns)
    
    # Check for new topic patterns
    has_new_topic_pattern = any(pattern in content_lower for pattern in new_topic_patterns)
    
    # If it has new topic patterns, it's likely NOT a follow-up
    if has_new_topic_pattern:
        return False
    
    # If it has follow-up patterns, it's likely a follow-up
    if has_followup_pattern:
        return True
    
    # Check if the content references data that would be in the context
    if last_tool_context and isinstance(last_tool_context, dict):
        # Look for references to data fields or values that might be in the context
        context_str = str(last_tool_context).lower()
        # Simple heuristic: if the question contains words that appear in the context
        content_words = set(content_lower.split())
        context_words = set(context_str.split())
        common_words = content_words.intersection(context_words)
        # Filter out common words
        meaningful_common = [w for w in common_words if len(w) > 3 and w not in ['the', 'and', 'for', 'with']]
        if len(meaningful_common) > 0:
            return True
    
    # Default to not a follow-up if uncertain
    return False

async def process_tool_response(called_tool_name, full_messages, message_history, 
                               markdown_links, actions, response_can_be_visualized):
    """Process tool response and set up actions"""
    if called_tool_name == "SemanticSearch":
        extracted_context, document_urls = extract_tool_context(full_messages)
        
        if document_urls:
            for i, url in enumerate(document_urls, 1):
                try:
                    parsed_url = urlparse(url)
                    domain = parsed_url.netloc or f"Source {i}"
                    markdown_links.append(f"[{domain}]({url})")
                except:
                    markdown_links.append(f"Source {i} [{url}]")
        
        if extracted_context:
            actions.extend([
                cl.Action(name="elaborate", icon="sparkles", label="Elaborate Further", 
                         payload={"context": extracted_context}),
                cl.Action(name="answer_followup", icon="message-circle-reply", 
                         label=generate_followup_question(extracted_context),
                         payload={"context": extracted_context})
            ])
    
    else:
        # Handle other tools (Vantage Services, etc.)
        extracted_context = extract_tool_context(full_messages)
        if extracted_context:
            cl.user_session.set("vantage_services_latest_context", extracted_context)
            response_can_be_visualized = True
            actions.extend([
                cl.Action(name="visualize_data", label="📊 Visualize Data", 
                         payload={"context": extracted_context}),
                cl.Action(name="elaborate", icon="sparkles", label="Elaborate Further", 
                         payload={"context": extracted_context}),
                cl.Action(name="answer_followup", icon="message-circle-reply", 
                         label=generate_followup_question(extracted_context, is_tabular_data=True),
                         payload={"context": extracted_context})
            ])

async def send_final_response(response, markdown_links, passed_content, actions):
    """Send the final response to user"""
    if len(markdown_links) > 0:
        response_message_content = response['messages'][-1].content + "\n**References**\n" + " | ".join(markdown_links)
        response_message_content = scan_response(response_message_content)
        response_message_content += f"\n\n**For the devs:**\n```{passed_content}```\n"
    else:
        response_message_content = response['messages'][-1].content
        response_message_content += f"\n\n**For the devs:**\n```{passed_content}```\n"

    await stream_response_to_user(response_message_content, actions)
```

## Key Enhancements:

1. **Smart Follow-up Detection**: Uses heuristics to determine if a question is likely a follow-up before running the agent
2. **Single Agent Call**: Only runs the agent once in most cases
3. **Context Validation**: Even if heuristics suggest follow-up, validates by checking if tools were actually used
4. **Cleaner Logic Flow**: Separates follow-up handling from normal processing
5. **Helper Functions**: Modular code for better maintainability

This approach eliminates the double tool call issue while maintaining the follow-up functionality you want.
