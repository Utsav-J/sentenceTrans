Looking at your code, I can see the follow-up logic is partially implemented but there are some issues with the current approach. Let me help you implement this feature properly.

The main issues I see are:

1. The `is_followup` variable is set but not properly used to maintain the follow-up state
2. The context refresh logic needs to be clearer about when to exit follow-up mode
3. The follow-up state should persist across multiple questions until a new tool call is made

Here's my implementation strategy:

## Key Changes Needed:

1. **Add persistent follow-up state tracking**
2. **Implement tool call detection for exiting follow-up mode**
3. **Maintain consistent context during follow-up**
4. **Clear follow-up state when new tool calls are made**

## Updated Implementation:

```python
@cl.on_message
async def main(message: cl.Message):
    """Handle incoming messages"""
    connection_info = cl.user_session.get("connection_info")
    
    if not connection_info or 'agent' not in connection_info:
        await cl.Message(content="❌ Agent not initialized. Please refresh the page.").send()
        return
    
    # Show typing indicator
    async with cl.Step(name="thinking", type="run") as step:
        step.output = "Processing your message..."
        try:
            # Get conversation history and follow-up state
            message_history = cl.user_session.get("message_history", [])
            is_followup = cl.user_session.get("is_followup", False)
            last_tool_context = cl.user_session.get("last_tool_context", None)
            
            if not message_history:
                message_history.append({"role": "system", "content": SYSTEM_PROMPT})
            
            # Add current user message
            message_history.append({"role": "user", "content": message.content})
            
            agent = connection_info['agent']
            
            # First, check if this query would require a tool call by running it
            response = await agent.ainvoke({"messages": message_history})
            full_messages = response.get("messages", []) if isinstance(response, dict) else []
            
            # Check if a new tool call was made
            new_tool_message = next((m for m in full_messages if isinstance(m, ToolMessage)), None)
            
            if new_tool_message:
                # NEW TOOL CALL DETECTED - Exit follow-up mode and refresh context
                step.output = "New tool call detected - refreshing context..."
                
                # Reset follow-up state
                cl.user_session.set("is_followup", False)
                
                # Create fresh context with just system prompt and current message
                fresh_history = [
                    {"role": "system", "content": SYSTEM_PROMPT},
                    {"role": "user", "content": message.content}
                ]
                
                # Re-run with fresh context
                response = await agent.ainvoke({"messages": fresh_history})
                full_messages = response.get("messages", []) if isinstance(response, dict) else []
                
                # Store new tool context
                new_tool_message = next((m for m in full_messages if isinstance(m, ToolMessage)), None)
                if new_tool_message:
                    try:
                        tool_content = json.loads(new_tool_message.content)
                        cl.user_session.set("last_tool_context", tool_content)
                        cl.user_session.set("is_followup", True)  # Enable follow-up mode
                    except Exception:
                        cl.user_session.set("last_tool_context", new_tool_message.content)
                        cl.user_session.set("is_followup", True)
                
                # Update message history with fresh context
                cl.user_session.set("message_history", fresh_history)
                
                await cl.Message(content="🔄 New data request detected - context refreshed!").send()
                
            elif is_followup and last_tool_context:
                # FOLLOW-UP MODE - Use previous tool context
                step.output = "Follow-up mode - using previous context..."
                
                await cl.Message(content="💡 Using previous data to answer your follow-up question.").send()
                
                # Create enhanced messages with previous tool context
                enhanced_messages = [{"role": "system", "content": SYSTEM_PROMPT}]
                
                # Add the previous tool context
                if isinstance(last_tool_context, dict):
                    json_str = json.dumps(last_tool_context, indent=2)
                else:
                    json_str = str(last_tool_context)
                
                enhanced_messages.append({
                    "role": "system",
                    "content": (
                        "You are answering a follow-up question using previous data context. "
                        "Here is the previous tool result data:\n\n"
                        f"{json_str}\n\n"
                        "Use this data to answer the user's current question."
                    )
                })
                enhanced_messages.append({"role": "user", "content": message.content})
                
                # Generate response using previous context
                response = await agent.ainvoke({"messages": enhanced_messages})
                
                # Keep follow-up mode active
                cl.user_session.set("is_followup", True)
                
            else:
                # NORMAL MODE - No tool call, no follow-up context
                step.output = "Normal response mode..."
                
                # Reset follow-up state if we had it
                if is_followup:
                    cl.user_session.set("is_followup", False)
                    cl.user_session.set("last_tool_context", None)
            
            # Handle different response types (FX data, RAG context, etc.)
            if full_messages:
                # Check for FX tool data
                fx_tool_message = next((m for m in full_messages if isinstance(m, ToolMessage) and 
                                     getattr(m, 'name', None) == 'GetForeignExchangeTransactionData'), None)
                
                if fx_tool_message:
                    system_message = enhance_tool_context_json(full_messages)
                    if system_message:
                        enhanced_messages = message_history.copy()
                        enhanced_messages.append(system_message)
                        final_response = await agent.ainvoke({"messages": enhanced_messages})
                        response = final_response
                    step.output = "Response generated with FX table context!"
                
                else:
                    # Handle RAG or other tool contexts
                    extracted_context, document_urls = extract_tool_context(full_messages)
                    if extracted_context:
                        context_msg = cl.Message(
                            content=f"**📚 Retrieved Context:**\n\n{extracted_context}",
                            author="System"
                        )
                        await context_msg.send()
                        
                        enhanced_messages = enhance_message_with_context(
                            message_history, extracted_context, document_urls
                        )
                        final_response = await agent.ainvoke({"messages": enhanced_messages})
                        response = final_response
                        step.output = "Response generated with enhanced context!"
            
            # Update message history
            message_history.append({"role": "assistant", "content": str(response)})
            cl.user_session.set("message_history", message_history)
            
        except Exception as e:
            step.output = f"Error: {str(e)}"
            response = f"❌ Sorry, I encountered an error: {str(e)}"
            print(f"Message processing error: {e}")
    
    # Send final response
    if isinstance(response, dict) and 'messages' in response:
        await cl.Message(content=str(response['messages'][-1].content)).send()
    else:
        await cl.Message(content=str(response)).send()
```

## Questions for you:

1. **Tool Call Detection**: How do you want to identify when a query "requires a tool call"? Should I:
   - Run the agent first and check if tools were used?
   - Pre-analyze the query text for keywords that suggest tool usage?
   - Use a different approach?

2. **Context Persistence**: When in follow-up mode, should the conversation history:
   - Keep growing with each follow-up question?
   - Stay minimal with just system + tool context + current question?
   - Have a maximum length before trimming?

3. **Follow-up Exit Conditions**: Besides new tool calls, should follow-up mode exit when:
   - User asks completely unrelated questions?
   - After a certain number of follow-up questions?
   - User explicitly says "new topic" or similar?

4. **Visual Indicators**: Would you like to show users when they're in follow-up mode with visual indicators or messages?step.output

Let me know your preferences on these points and I can refine the implementation further!

Perfect! Based on your requirements, here's the refined implementation:

```python
@cl.on_message
async def main(message: cl.Message):
    """Handle incoming messages"""
    connection_info = cl.user_session.get("connection_info")
    
    if not connection_info or 'agent' not in connection_info:
        await cl.Message(content="❌ Agent not initialized. Please refresh the page.").send()
        return
    
    # Show typing indicator
    async with cl.Step(name="thinking", type="run") as step:
        step.output = "Processing your message..."
        try:
            # Get conversation history and follow-up state
            message_history = cl.user_session.get("message_history", [])
            is_followup = cl.user_session.get("is_followup", False)
            last_tool_context = cl.user_session.get("last_tool_context", None)
            
            if not message_history:
                message_history.append({"role": "system", "content": SYSTEM_PROMPT})
            
            # Add current user message to history
            message_history.append({"role": "user", "content": message.content})
            
            agent = connection_info['agent']
            
            # Run the agent to check if tools are used
            response = await agent.ainvoke({"messages": message_history})
            full_messages = response.get("messages", []) if isinstance(response, dict) else []
            
            # Check if a new tool call was made
            new_tool_message = next((m for m in full_messages if isinstance(m, ToolMessage)), None)
            
            if new_tool_message:
                # NEW TOOL CALL DETECTED - Refresh context and enable follow-up mode
                step.output = "Processing with tools..."
                
                # Store new tool context
                try:
                    tool_content = json.loads(new_tool_message.content)
                except Exception:
                    tool_content = new_tool_message.content
                
                cl.user_session.set("last_tool_context", tool_content)
                cl.user_session.set("is_followup", True)
                
                # Process the response normally (with tool enhancements)
                
            elif is_followup and last_tool_context:
                # FOLLOW-UP MODE - Use previous tool context
                step.output = "Using previous context for follow-up..."
                
                # Create enhanced messages with previous tool context
                enhanced_messages = message_history.copy()
                
                # Add the previous tool context as system message
                if isinstance(last_tool_context, dict):
                    json_str = json.dumps(last_tool_context, indent=2)
                else:
                    json_str = str(last_tool_context)
                
                context_system_msg = {
                    "role": "system",
                    "content": (
                        "Previous tool data is available for context. "
                        "Use this data if relevant to the current question:\n\n"
                        f"{json_str}"
                    )
                }
                enhanced_messages.append(context_system_msg)
                
                # Re-run with enhanced context
                response = await agent.ainvoke({"messages": enhanced_messages})
                full_messages = response.get("messages", []) if isinstance(response, dict) else []
                
                # Keep follow-up mode active (don't change is_followup)
                
            else:
                # NORMAL MODE - No tool call, no follow-up context
                step.output = "Processing normally..."
                
                # If we were in follow-up mode but no longer relevant, exit it
                if is_followup:
                    cl.user_session.set("is_followup", False)
                    cl.user_session.set("last_tool_context", None)
            
            # Handle different response types (FX data, RAG context, etc.)
            if full_messages:
                # Check for FX tool data
                fx_tool_message = next((m for m in full_messages if isinstance(m, ToolMessage) and 
                                     getattr(m, 'name', None) == 'GetForeignExchangeTransactionData'), None)
                
                if fx_tool_message:
                    system_message = enhance_tool_context_json(full_messages)
                    if system_message:
                        enhanced_messages = message_history.copy()
                        enhanced_messages.append(system_message)
                        final_response = await agent.ainvoke({"messages": enhanced_messages})
                        response = final_response
                    step.output = "Response generated with FX data!"
                
                else:
                    # Handle RAG or other tool contexts
                    extracted_context, document_urls = extract_tool_context(full_messages)
                    if extracted_context:
                        context_msg = cl.Message(
                            content=f"**📚 Retrieved Context:**\n\n{extracted_context}",
                            author="System"
                        )
                        await context_msg.send()
                        
                        enhanced_messages = enhance_message_with_context(
                            message_history, extracted_context, document_urls
                        )
                        final_response = await agent.ainvoke({"messages": enhanced_messages})
                        response = final_response
                        step.output = "Response generated with context!"
            
            # Add assistant response to growing message history
            if isinstance(response, dict) and 'messages' in response:
                assistant_content = str(response['messages'][-1].content)
            else:
                assistant_content = str(response)
                
            message_history.append({"role": "assistant", "content": assistant_content})
            cl.user_session.set("message_history", message_history)
            
        except Exception as e:
            step.output = f"Error: {str(e)}"
            assistant_content = f"❌ Sorry, I encountered an error: {str(e)}"
            # Still add error to history
            message_history.append({"role": "assistant", "content": assistant_content})
            cl.user_session.set("message_history", message_history)
            print(f"Message processing error: {e}")
    
    # Send final response
    await cl.Message(content=assistant_content).send()
```

## Key Changes Made:

1. **Tool Detection**: The agent runs first, then we check if any `ToolMessage` exists in the response to detect tool usage.

2. **Conversation History**: The `message_history` keeps growing with each question and answer, maintaining full conversational context.

3. **Follow-up Logic**: 
   - When a tool call is made → `is_followup = True` and store `last_tool_context`
   - When in follow-up mode → Add previous tool context as a system message to help answer follow-up questions
   - When unrelated questions are asked → The agent naturally won't use the previous tool context, and follow-up mode remains active until a new tool call

4. **No Visual Indicators**: Removed all the messages like "🔄 Context refreshed" and "💡 Using previous data".

5. **Exit Condition**: Follow-up mode only exits when a new tool call is made, not when unrelated questions are asked (the context just becomes irrelevant naturally).

The logic now works as:
- **First tool call** → Enable follow-up mode, store tool context
- **Follow-up questions** → Use previous tool context + full conversation history
- **Unrelated questions** → Still in follow-up mode, but previous context becomes irrelevant naturally
- **New tool call** → New tool context replaces old one, follow-up continues with new context

This should give you the behavior you're looking for!
