Thanks for the image — you’re working in a LangGraph + Tachyon Agent setup with tachyon_a2a_core, and it looks like you want to integrate MCP server communication using those modules and existing conventions.

Here’s how you can create a LangGraph-compatible agent using your existing dependencies that communicates with your MCP server running at http://localhost:8081/mcp using StreamableHttp.


---

✅ Step-by-Step Integration (with your imports)

1. Create an MCP Tool using StreamableHttpChannel

from google.adk.clients import StreamableHttpChannel
from langchain_core.tools import tool


@tool
async def mcp_proxy_tool(input: str) -> str:
    """
    Tool to send input to MCP server and return its output via Streamable HTTP.
    """
    channel = StreamableHttpChannel(url="http://localhost:8081/mcp")
    async with channel:
        await channel.send({"input": input})
        response = await channel.receive()
        return response.get("output", "No output received.")

> You can later rename this or register multiple tools if your MCP server handles more than one route/tool.




---

2. Wrap Tool in LangGraph Agent

You're already importing create_react_agent, ToolExecutor, and AIMessage, so use them to define the graph.

from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolExecutor, create_react_agent
from langchain_core.messages import HumanMessage, AIMessage

# Step 1: Tool Executor
tools = [mcp_proxy_tool]
tool_executor = ToolExecutor(tools)

# Step 2: Create the agent using ReAct
agent_runnable = create_react_agent(tool_executor)

# Step 3: Define LangGraph
builder = StateGraph()
builder.add_node("agent", agent_runnable)
builder.set_entry_point("agent")
builder.add_edge("agent", END)

# Step 4: Compile the graph
graph = builder.compile()


---

3. Run the Agent

import asyncio

async def main():
    response = await graph.ainvoke({
        "messages": [HumanMessage(content="Hello MCP agent!")]
    })
    print("\n=== MCP Agent Final Output ===")
    print(response)

if __name__ == "__main__":
    asyncio.run(main())


---

✅ Final Notes

This integrates LangGraph, Tachyon tools, and MCP over Streamable HTTP, all while respecting the structure and imports shown in your image.

If your MCP tool expects a different input schema (e.g. "message": ... instead of "input"), adjust this line:

await channel.send({"input": input})

If you’re using tachyon_a2a_core.agents.base_agent.BaseAgent, and want to define an actual A2A Agent that talks to MCP, I can scaffold that too.



---

Would you like this agent wrapped in a BaseAgent subclass using your tachyon_a2a_core imports?

