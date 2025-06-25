import chainlit as cl
from dotenv import load_dotenv
from typing import Literal
from langchain_core.messages import HumanMessage, SystemMessage
from langchain.schema.runnable.config import RunnableConfig
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langgraph.prebuilt import ToolNode

from mcp.client.streamable_http import streamableHttpClient
from mcp import ClientSession
from langchain_mcp_adapters.tools import create_react_agent
from tachyon_langchain_client import TachyonLangchainClient
from langchain.prebuilt import load_mcp_tools

load_dotenv()

SERVER_URL = "http://localhost:8081/mcp"

# Chainlit Chat Start
@cl.on_chat_start
async def on_chat_start():
    await cl.Message(content="⚙️ Initializing Vantage agent...").send()

    stream_ctx = streamableHttpClient(SERVER_URL)
    stream = await stream_ctx.__aenter__()
    read, write, _ = stream
    session = ClientSession(read, write)
    await session.initialize()

    tools = await load_mcp_tools(session)
    model = TachyonLangchainClient(model_name="gemini-2.0-flash")
    model = model.bind_tools(tools)
    final_model = model.with_config(tags=["final_node"])

    # Store context for later use
    cl.user_session.set("mcp_stream_ctx", stream_ctx)
    cl.user_session.set("model", model)
    cl.user_session.set("final_model", final_model)
    cl.user_session.set("tools", tools)

    await cl.Message(content="✅ Vantage Agent is ready. Ask your question!").send()

# LangGraph nodes
def should_continue(state: MessagesState) -> Literal["tools", "final"]:
    messages = state["messages"]
    if messages and messages[-1].tool_calls:
        return "tools"
    return "final"

def call_model(state: MessagesState):
    model = cl.user_session.get("model")
    messages = state["messages"]
    response = model.invoke(messages)
    return {"messages": [response]}

def call_final_model(state: MessagesState):
    final_model = cl.user_session.get("final_model")
    messages = state["messages"]
    last_ai = messages[-1]
    response = final_model.invoke([
        SystemMessage(content="Respond in a conversational, helpful tone."),
        HumanMessage(content=last_ai.content),
    ])
    response.id = last_ai.id  # preserve message ID
    return {"messages": [response]}

@cl.on_message
async def on_message(msg: cl.Message):
    tools = cl.user_session.get("tools")
    model = cl.user_session.get("model")

    # Build LangGraph dynamically on message (or cache if preferred)
    builder = StateGraph(MessagesState)
    builder.add_node("agent", call_model)
    builder.add_node("tools", ToolNode(tools=tools))
    builder.add_node("final", call_final_model)

    builder.add_edge(START, "agent")
    builder.add_conditional_edges("agent", should_continue)
    builder.add_edge("tools", "agent")
    builder.add_edge("final", END)

    graph = builder.compile()

    config = {
        "configurable": {"thread_id": cl.context.session.id}
    }
    cb = cl.LangchainCallbackHandler()
    final_answer = cl.Message(content="")

    async for output, metadata in graph.astream(
        {"messages": [HumanMessage(content=msg.content)]},
        stream_mode="messages",
        config=RunnableConfig(callbacks=[cb], **config)
    ):
        if (
            output.get("messages")
            and not isinstance(output["messages"][-1], HumanMessage)
            and metadata.get("langgraph_node") == "final"
        ):
            await final_answer.stream_token(output["messages"][-1].content)

    await final_answer.send()

@cl.on_chat_end
async def on_chat_end():
    stream_ctx = cl.user_session.get("mcp_stream_ctx")
    if stream_ctx:
        await stream_ctx.__aexit__(None, None, None)
