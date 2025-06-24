import asyncio
import json
from typing import Any, Dict, List, Optional, TypedDict, Annotated
from dataclasses import dataclass

import httpx
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage, ToolMessage
from langchain_core.tools import BaseTool
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode
from langgraph.graph.message import add_messages


class AgentState(TypedDict):
    messages: Annotated[List[BaseMessage], add_messages]


@dataclass
class MCPToolCall:
    name: str
    arguments: Dict[str, Any]


class MCPTool(BaseTool):
    """LangChain tool wrapper for MCP server tools"""
    
    def __init__(self, mcp_client: 'MCPClient', tool_info: Dict[str, Any]):
        self.mcp_client = mcp_client
        self.tool_info = tool_info
        super().__init__(
            name=tool_info['name'],
            description=tool_info.get('description', ''),
            args_schema=self._create_args_schema(tool_info)
        )
    
    def _create_args_schema(self, tool_info: Dict[str, Any]):
        """Create Pydantic schema from MCP tool schema"""
        from pydantic import BaseModel, create_model
        
        input_schema = tool_info.get('inputSchema', {})
        properties = input_schema.get('properties', {})
        required = input_schema.get('required', [])
        
        fields = {}
        for prop_name, prop_info in properties.items():
            prop_type = str  # Default to string, could be enhanced
            if prop_info.get('type') == 'integer':
                prop_type = int
            elif prop_info.get('type') == 'number':
                prop_type = float
            elif prop_info.get('type') == 'boolean':
                prop_type = bool
            
            if prop_name in required:
                fields[prop_name] = (prop_type, ...)
            else:
                fields[prop_name] = (Optional[prop_type], None)
        
        return create_model(f"{tool_info['name']}Args", **fields)
    
    async def _arun(self, **kwargs) -> str:
        """Execute the MCP tool"""
        return await self.mcp_client.call_tool(self.name, kwargs)
    
    def _run(self, **kwargs) -> str:
        """Sync wrapper for async execution"""
        return asyncio.run(self._arun(**kwargs))


class MCPClient:
    """HTTP client for MCP server communication"""
    
    def __init__(self, base_url: str = "http://localhost:8081/mcp"):
        self.base_url = base_url
        self.client = httpx.AsyncClient(timeout=30.0)
        self.tools: List[Dict[str, Any]] = []
        self.langchain_tools: List[MCPTool] = []
    
    async def initialize(self):
        """Initialize connection and load available tools"""
        try:
            # Initialize the MCP session
            init_response = await self.client.post(
                f"{self.base_url}/initialize",
                json={
                    "protocolVersion": "2024-11-05",
                    "capabilities": {
                        "tools": {}
                    },
                    "clientInfo": {
                        "name": "langgraph-mcp-client",
                        "version": "1.0.0"
                    }
                }
            )
            init_response.raise_for_status()
            print(f"MCP session initialized: {init_response.json()}")
            
            # List available tools
            await self.list_tools()
            
        except Exception as e:
            print(f"Failed to initialize MCP client: {e}")
            raise
    
    async def list_tools(self):
        """Retrieve and cache available tools from MCP server"""
        try:
            response = await self.client.post(
                f"{self.base_url}/tools/list",
                json={}
            )
            response.raise_for_status()
            
            result = response.json()
            self.tools = result.get('tools', [])
            
            # Create LangChain tool wrappers
            self.langchain_tools = [
                MCPTool(self, tool_info) for tool_info in self.tools
            ]
            
            print(f"Loaded {len(self.tools)} tools from MCP server:")
            for tool in self.tools:
                print(f"  - {tool['name']}: {tool.get('description', 'No description')}")
                
        except Exception as e:
            print(f"Failed to list tools: {e}")
            raise
    
    async def call_tool(self, tool_name: str, arguments: Dict[str, Any]) -> str:
        """Execute a tool on the MCP server"""
        try:
            response = await self.client.post(
                f"{self.base_url}/tools/call",
                json={
                    "name": tool_name,
                    "arguments": arguments
                }
            )
            response.raise_for_status()
            
            result = response.json()
            
            # Extract content from the response
            content = result.get('content', [])
            if content and isinstance(content, list):
                return '\n'.join([
                    item.get('text', str(item)) 
                    for item in content 
                    if isinstance(item, dict)
                ])
            
            return str(result)
            
        except Exception as e:
            print(f"Tool call failed for {tool_name}: {e}")
            return f"Error calling tool {tool_name}: {str(e)}"
    
    async def close(self):
        """Close the HTTP client"""
        await self.client.aclose()


class MCPLangGraphAgent:
    """LangGraph agent with MCP server integration"""
    
    def __init__(self, mcp_client: MCPClient, llm_model: str = "gpt-4"):
        self.mcp_client = mcp_client
        self.llm = ChatOpenAI(model=llm_model, temperature=0)
        self.graph = None
    
    def build_graph(self):
        """Build the LangGraph workflow"""
        
        # Create tool node with MCP tools
        tool_node = ToolNode(self.mcp_client.langchain_tools)
        
        # Define the agent node
        def agent_node(state: AgentState) -> AgentState:
            messages = state["messages"]
            
            # Bind tools to the LLM
            llm_with_tools = self.llm.bind_tools(self.mcp_client.langchain_tools)
            
            # Generate response
            response = llm_with_tools.invoke(messages)
            
            return {"messages": [response]}
        
        # Define conditional logic for routing
        def should_continue(state: AgentState) -> str:
            messages = state["messages"]
            last_message = messages[-1]
            
            # If the last message has tool calls, continue to tools
            if hasattr(last_message, 'tool_calls') and last_message.tool_calls:
                return "tools"
            
            # Otherwise, end the conversation
            return END
        
        # Build the graph
        workflow = StateGraph(AgentState)
        
        # Add nodes
        workflow.add_node("agent", agent_node)
        workflow.add_node("tools", tool_node)
        
        # Add edges
        workflow.add_edge(START, "agent")
        workflow.add_conditional_edges(
            "agent",
            should_continue,
            {"tools": "tools", END: END}
        )
        workflow.add_edge("tools", "agent")
        
        # Compile the graph
        self.graph = workflow.compile()
        
        return self.graph
    
    async def run(self, user_input: str) -> str:
        """Run the agent with user input"""
        if not self.graph:
            self.build_graph()
        
        # Create initial state
        initial_state = {
            "messages": [HumanMessage(content=user_input)]
        }
        
        # Run the graph
        result = await self.graph.ainvoke(initial_state)
        
        # Extract the final response
        messages = result["messages"]
        if messages:
            last_message = messages[-1]
            if isinstance(last_message, AIMessage):
                return last_message.content
        
        return "No response generated"


async def main():
    """Main function to demonstrate the MCP LangGraph agent"""
    
    # Initialize MCP client
    mcp_client = MCPClient("http://localhost:8081/mcp")
    
    try:
        # Initialize the MCP connection
        print("Initializing MCP client...")
        await mcp_client.initialize()
        
        # Create the LangGraph agent
        agent = MCPLangGraphAgent(mcp_client)
        agent.build_graph()
        
        print("\nMCP LangGraph Agent is ready!")
        print("Available tools:")
        for tool in mcp_client.tools:
            print(f"  - {tool['name']}: {tool.get('description', 'No description')}")
        
        # Interactive loop
        print("\nType 'quit' to exit")
        while True:
            user_input = input("\nYou: ").strip()
            
            if user_input.lower() in ['quit', 'exit', 'q']:
                break
            
            if not user_input:
                continue
            
            try:
                print("Assistant: ", end="", flush=True)
                response = await agent.run(user_input)
                print(response)
                
            except Exception as e:
                print(f"Error: {e}")
    
    except Exception as e:
        print(f"Failed to initialize agent: {e}")
    
    finally:
        # Clean up
        await mcp_client.close()
        print("\nGoodbye!")


# Alternative: Synchronous wrapper for easier usage
class SyncMCPLangGraphAgent:
    """Synchronous wrapper for the async MCP LangGraph agent"""
    
    def __init__(self, mcp_server_url: str = "http://localhost:8081/mcp", llm_model: str = "gpt-4"):
        self.mcp_server_url = mcp_server_url
        self.llm_model = llm_model
        self.agent = None
        self.mcp_client = None
    
    def initialize(self):
        """Initialize the agent synchronously"""
        async def _init():
            self.mcp_client = MCPClient(self.mcp_server_url)
            await self.mcp_client.initialize()
            self.agent = MCPLangGraphAgent(self.mcp_client, self.llm_model)
            self.agent.build_graph()
        
        asyncio.run(_init())
    
    def run(self, user_input: str) -> str:
        """Run the agent synchronously"""
        if not self.agent:
            raise RuntimeError("Agent not initialized. Call initialize() first.")
        
        return asyncio.run(self.agent.run(user_input))
    
    def close(self):
        """Close the agent synchronously"""
        if self.mcp_client:
            asyncio.run(self.mcp_client.close())


if __name__ == "__main__":
    # Run the async main function
    asyncio.run(main())
