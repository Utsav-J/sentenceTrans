from openai import OpenAI
from typing import List

client = OpenAI()  # Assumes `OPENAI_API_KEY` is set in the environment

def is_new_question(messages: List[str], model: str = "gpt-4") -> bool:
    """
    Uses an LLM to determine if the last message in the list is a new question.

    Parameters:
        messages (List[str]): List of user messages (strings).
        model (str): OpenAI model to use.

    Returns:
        bool: True if the latest message is a new question, False if it's a follow-up.
    """
    if len(messages) < 2:
        return True

    # Construct prompt for the LLM
    system_prompt = (
        "You are an AI assistant that helps identify whether the last user message is a completely new "
        "question or if it's a follow-up or related to the previous conversation.\n"
        "Reply with only `true` (if it's a new question) or `false` (if it's a follow-up or related)."
    )

    # Create a message list for the chat completion
    chat = [{"role": "system", "content": system_prompt}]
    for msg in messages[:-1]:
        chat.append({"role": "user", "content": msg})
    chat.append({"role": "user", "content": f"(Latest message): {messages[-1]}"})

    # LLM call
    response = client.chat.completions.create(
        model=model,
        messages=chat,
        temperature=0,
    )

    output = response.choices[0].message.content.strip().lower()
    return output == "true"
