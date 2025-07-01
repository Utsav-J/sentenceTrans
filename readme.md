Perfect! Here's the implementation with the persistent follow-up mode:

```python
@cl.on_message
async def main(message: cl.Message):
    """Handle incoming messages"""
    connection_info = cl.user_session.get("connection_info")
    
    if not connection_info or 'agent' not in connection_info:
        await cl.Message(content="❌ Agent not initialized. Please refresh the page.").send()
        return
    
    # Get persistent follow-up state
    is_in_followup_mode = cl.user_session.get("is_in_followup_mode", False)
    
    # Show typing indicator with follow-up status
    indicator_text = "🔄 Processing your follow-up question..." if is_in_followup_mode else "Processing your message..."
    async with cl.Step(name="thinking", type="run") as step:
        step.output = indicator_text
        passed_content = classify_intent(message.content.strip())
        try:
            # Get conversation history
            message_history = cl.user_session.get("message_history", [])
            if not message_history:
                message_history.append({"role": "system", "content": SYSTEM_PROMPT})
            message_history.append({"role": "user", "content": message.content})
            
            # If in follow-up mode, inject cached context before making agent call
            if is_in_followup_mode:
                last_tool_context = cl.user_session.get("last_tool_context", None)
                if last_tool_context:
                    # Add cached context to the current query
                    enhanced_messages = message_history.copy()
                    json_str = json.dumps(last_tool_context, indent=2) if isinstance(last_tool_context, dict) else str(last_tool_context)
                    enhanced_messages.insert(-1, {  # Insert before the current user message
                        "role": "system",
                        "content": (
                            "You are in follow-up mode. Here is the previous data context (in JSON):\n\n"
                            f"{json_str}\n\nUse this data context to help answer the current question."
                        )
                    })
                    agent = connection_info['agent']
                    response = await agent.ainvoke({"messages": enhanced_messages})
                else:
                    agent = connection_info['agent']
                    response = await agent.ainvoke({"messages": message_history})
            else:
                agent = connection_info['agent']
                response = await agent.ainvoke({"messages": message_history})
            
            full_messages = response.get("messages", []) if isinstance(response, dict) else []

            # --- MEMORY REFRESH & FOLLOW-UP LOGIC START ---
            first_tool_message = next((m for m in full_messages if isinstance(m, ToolMessage)), None)
            new_tool_call_signature = None
            tool_context_to_store = None
            
            if first_tool_message:
                tool_name = getattr(first_tool_message, 'name', None)
                try:
                    tool_content = json.loads(first_tool_message.content)
                except Exception:
                    tool_content = first_tool_message.content
                if isinstance(tool_content, dict):
                    params = tuple(sorted((k, str(v)) for k, v in tool_content.items() if k != 'result'))
                else:
                    params = tuple()
                new_tool_call_signature = (tool_name, params)
                tool_context_to_store = tool_content
            
            last_tool_call_signature = cl.user_session.get("last_tool_call", None)
            context_reset = False
            
            if new_tool_call_signature and new_tool_call_signature != last_tool_call_signature:
                # Completely new tool call signature detected - reset context and exit follow-up mode
                context_reset = True
                cl.user_session.set("last_tool_call", new_tool_call_signature)
                cl.user_session.set("last_tool_context", tool_context_to_store)
                cl.user_session.set("is_in_followup_mode", False)  # Reset follow-up mode
                
                message_history = [{"role": "system", "content": SYSTEM_PROMPT}, {"role": "user", "content": message.content}]
                cl.user_session.set("message_history", message_history)
                
                await cl.Message(content="🔄 Context has been refreshed due to a new topic or data request. Exiting follow-up mode.").send()
                
                response = await agent.ainvoke({"messages": message_history})
                full_messages = response.get("messages", []) if isinstance(response, dict) else []
                
            elif new_tool_call_signature:
                # Same tool call signature - stay in current mode, just update context
                cl.user_session.set("last_tool_call", new_tool_call_signature)
                cl.user_session.set("last_tool_context", tool_context_to_store)
            
            # --- MEMORY REFRESH & FOLLOW-UP LOGIC END ---

            # Check if we should enter follow-up mode (after processing tool calls)
            if not is_in_followup_mode and not context_reset and cl.user_session.get("last_tool_context"):
                # We have tool context and we're not already in follow-up mode - enter it
                cl.user_session.set("is_in_followup_mode", True)
                await cl.Message(content="💬 **Follow-up mode activated!** Your next questions will use the current data context. Ask a new data query to exit this mode.", author="System").send()
            elif is_in_followup_mode and not context_reset:
                # We're continuing in follow-up mode
                await cl.Message(content="💬 **Still in follow-up mode** - using previous data context for your question.", author="System").send()

            # Process the response based on tool messages
            if not full_messages:
                message_history.append({"role": "assistant", "content": str(response)})
                cl.user_session.set("message_history", message_history)
                step.output = "Response generated!" + (" (Follow-up mode)" if cl.user_session.get("is_in_followup_mode") else "")
            else:
                fx_tool_message = next((m for m in full_messages if isinstance(m, ToolMessage) and getattr(m, 'name', None) == 'GetForeignExchangeTransactionData'), None)
                if fx_tool_message:
                    system_message = enhance_tool_context_json(full_messages)
                    print("got json data\n\n\n\n")
                    enhanced_messages = message_history.copy()
                    if system_message:
                        enhanced_messages.append(system_message)
                    final_response = await agent.ainvoke({"messages": enhanced_messages})
                    response = final_response
                    message_history.append({"role": "assistant", "content": str(response)})
                    cl.user_session.set("message_history", message_history)
                    step.output = "Response generated with FX table context!" + (" (Follow-up mode)" if cl.user_session.get("is_in_followup_mode") else "")
                else:
                    extracted_context, document_urls = extract_tool_context(full_messages)
                    if extracted_context and not is_in_followup_mode:  # Only show context if not in follow-up mode
                        context_msg = cl.Message(
                            content=f"**📚 Retrieved Context:**\n\n{extracted_context}",
                            author="System"
                        )
                        await context_msg.send()
                    enhanced_messages = enhance_message_with_context(
                        message_history, extracted_context, document_urls
                    )
                    if extracted_context:
                        final_response = await agent.ainvoke({"messages": enhanced_messages})
                        response = final_response
                    message_history.append({"role": "assistant", "content": str(response)})
                    cl.user_session.set("message_history", message_history)
                    step.output = "Response generated with enhanced context!" + (" (Follow-up mode)" if cl.user_session.get("is_in_followup_mode") else "")
                    
        except Exception as e:
            step.output = f"Error: {str(e)}"
            response = f"❌ Sorry, I encountered an error: {str(e)}"
            print(f"Message processing error: {e}")
    
    await cl.Message(content=str(response['messages'][-1].content)).send()
```

**Key Changes Made:**

1. **Persistent Flag**: Added `is_in_followup_mode` flag stored in user session
2. **Context Injection**: When in follow-up mode, cached context is injected before the agent call
3. **Smart Reset**: Only resets follow-up mode when a completely different tool call signature is detected
4. **Visual Indicators**: 
   - Different processing messages based on mode
   - Clear notifications when entering/staying in follow-up mode
   - Status indicators in step outputs
5. **Mode Management**: Automatically enters follow-up mode after tool calls and exits only on new tool signatures

The system now maintains follow-up state across multiple messages until a genuinely new tool call is made!
