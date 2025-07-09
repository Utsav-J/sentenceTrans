# test_tool_correctness.py

from deepeval import evaluate
from deepeval.test_case import LLMTestCase, ToolCall
from deepeval.metrics import ToolCorrectnessMetric

def test_agent_tool_calls():
    # Simulate agent behavior in a test scenario
    # Replace with actual input/output/tool-calls from your agent
    input_text = "Find the weather in Bengaluru right now"
    # Suppose your agent called WebSearch and WeatherTool
    actual_output = agent.run(input_text)  # or paste agent output manually
    
    # Mock tool call trace (ensure names match your agent's tool IDs)
    tools_called = [
        ToolCall(name="WebSearch", input_parameters={"query": "Bengaluru weather"}),
        ToolCall(name="WeatherTool", input_parameters={"location": "Bengaluru"})
    ]
    
    # Define the expected tools your agent should use
    expected_tools = [
        ToolCall(name="WebSearch"),
        ToolCall(name="WeatherTool")
    ]
    
    test_case = LLMTestCase(
        input=input_text,
        actual_output=actual_output,
        tools_called=tools_called,
        expected_tools=expected_tools
    )
    
    metric = ToolCorrectnessMetric(
        include_reason=True,
        evaluation_params=[]             # default: only checks tool names
    )
    
    results = evaluate(test_cases=[test_case], metrics=[metric])
    
    score = results[0].metrics["ToolCorrectnessMetric"].score
    reason = results[0].metrics["ToolCorrectnessMetric"].reason
    
    assert score == 1.0, f"Tool correctness failed: {reason}"
