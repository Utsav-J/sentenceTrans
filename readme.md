import asyncio
import chainlit as cl
from mcp.client.streamable_http import streamableHttpClient
from dotenv import load_dotenv
from mcp import ClientSession
from langchain.prebuilt import load_mcp_tools
from langchain_mcp_adapters.tools import create_react_agent
from tachyon_langchain_client import TachyonLangchainClient

load_dotenv()

server_url = "http://localhost:8081/mcp"
model_client = TachyonLangchainClient(model_name="gemini-2.0-flash")

# Global variables to store session and agent
session = None
agent = None

@cl.on_chat_start
async def start():
    """Initialize the MCP session and agent when chat starts"""
    global session, agent
    
    # Show loading message
    msg = cl.Message(content="🔧 Initializing Vantage Chat Agent...")
    await msg.send()
    
    try:
        # Initialize MCP connection
        read, write, _ = await streamableHttpClient(server_url).__aenter__()
        session = await ClientSession(read, write).__aenter__()
        await session.initialize()
        
        # Load tools and create agent
        tools = await load_mcp_tools(session)
        agent = create_react_agent(model_client, tools)
        
        # Update message to show ready state
        await msg.update(content="✅ Vantage Chat Agent is ready! Ask me anything.")
        
    except Exception as e:
        await msg.update(content=f"❌ Failed to initialize agent: {str(e)}")
        raise

@cl.on_message
async def main(message: cl.Message):
    """Handle incoming messages"""
    global agent
    
    if agent is None:
        await cl.Message(content="❌ Agent not initialized. Please refresh the page.").send()
        return
    
    # Show typing indicator
    async with cl.Step(name="thinking", type="run") as step:
        step.output = "Processing your message..."
        
        try:
            # Get conversation history from Chainlit
            message_history = cl.user_session.get("message_history", [])
            
            # Add current user message
            message_history.append({"role": "user", "content": message.content})
            
            # Get agent response
            response = await agent.ainvoke({"messages": message_history})
            
            # Add assistant response to history
            message_history.append({"role": "assistant", "content": str(response)})
            
            # Store updated history
            cl.user_session.set("message_history", message_history)
            
            step.output = "Response generated!"
            
        except Exception as e:
            step.output = f"Error: {str(e)}"
            response = f"❌ Sorry, I encountered an error: {str(e)}"
    
    # Send the response
    await cl.Message(content=str(response)).send()

@cl.on_chat_end
async def end():
    """Clean up when chat ends"""
    global session
    
    if session:
        try:
            await session.__aexit__(None, None, None)
        except Exception as e:
            print(f"Error closing session: {e}")

if __name__ == "__main__":
    # Run the Chainlit app
    cl.run()
