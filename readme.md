import asyncio
import chainlit as cl
from mcp.client.streamable_http import streamableHttpClient
from dotenv import load_dotenv
from mcp import ClientSession
from langchain.prebuilt import load_mcp_tools
from langchain_mcp_adapters.tools import create_react_agent
from tachyon_langchain_client import TachyonLangchainClient
import weakref

load_dotenv()

server_url = "http://localhost:8081/mcp"
model_client = TachyonLangchainClient(model_name="gemini-2.0-flash")

# Store active connections per session
active_connections = {}

async def create_mcp_session():
    """Create and initialize MCP session with proper error handling"""
    try:
        # Create the HTTP client
        http_client = streamableHttpClient(server_url)
        read, write, close_func = await http_client.__aenter__()
        
        # Create the client session
        client_session = ClientSession(read, write)
        session = await client_session.__aenter__()
        await session.initialize()
        
        # Load tools and create agent
        tools = await load_mcp_tools(session)
        agent = create_react_agent(model_client, tools)
        
        return {
            'http_client': http_client,
            'client_session': client_session,
            'session': session,
            'agent': agent,
            'close_func': close_func
        }
    except Exception as e:
        print(f"Error creating MCP session: {e}")
        raise

async def cleanup_connection(connection_info):
    """Safely cleanup MCP connection"""
    try:
        if 'client_session' in connection_info:
            await connection_info['client_session'].__aexit__(None, None, None)
        if 'http_client' in connection_info:
            await connection_info['http_client'].__aexit__(None, None, None)
        if 'close_func' in connection_info:
            await connection_info['close_func']()
    except Exception as e:
        print(f"Error during cleanup: {e}")

@cl.on_chat_start
async def start():
    """Initialize the MCP session and agent when chat starts"""
    # Show loading message
    msg = cl.Message(content="🔧 Initializing Vantage Chat Agent...")
    await msg.send()
    
    try:
        # Create MCP session
        connection_info = await create_mcp_session()
        
        # Store connection info in user session
        cl.user_session.set("connection_info", connection_info)
        cl.user_session.set("message_history", [])
        
        # Store in global dict for cleanup (using session id as key)
        session_id = cl.user_session.get("id")
        active_connections[session_id] = connection_info
        
        # Update message to show ready state
        await msg.update(content="✅ Vantage Chat Agent is ready! Ask me anything.")
        
    except Exception as e:
        await msg.update(content=f"❌ Failed to initialize agent: {str(e)}")
        print(f"Initialization error: {e}")

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
            # Get conversation history
            message_history = cl.user_session.get("message_history", [])
            
            # Add current user message
            message_history.append({"role": "user", "content": message.content})
            
            # Get agent response
            agent = connection_info['agent']
            response = await agent.ainvoke({"messages": message_history})
            
            # Add assistant response to history
            message_history.append({"role": "assistant", "content": str(response)})
            
            # Store updated history
            cl.user_session.set("message_history", message_history)
            
            step.output = "Response generated!"
            
        except Exception as e:
            step.output = f"Error: {str(e)}"
            response = f"❌ Sorry, I encountered an error: {str(e)}"
            print(f"Message processing error: {e}")
    
    # Send the response
    await cl.Message(content=str(response)).send()

@cl.on_chat_end
async def end():
    """Clean up when chat ends"""
    session_id = cl.user_session.get("id")
    
    if session_id in active_connections:
        connection_info = active_connections[session_id]
        await cleanup_connection(connection_info)
        del active_connections[session_id]

@cl.on_stop
async def stop():
    """Clean up all connections when server stops"""
    cleanup_tasks = []
    for connection_info in active_connections.values():
        cleanup_tasks.append(cleanup_connection(connection_info))
    
    if cleanup_tasks:
        await asyncio.gather(*cleanup_tasks, return_exceptions=True)
    
    active_connections.clear()

if __name__ == "__main__":
    try:
        cl.run()
    except KeyboardInterrupt:
        print("Shutting down...")
    finally:
        # Ensure cleanup on exit
        if active_connections:
            loop = asyncio.get_event_loop()
            if loop.is_running():
                for connection_info in active_connections.values():
                    asyncio.create_task(cleanup_connection(connection_info))
            else:
                asyncio.run(stop())
