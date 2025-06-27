FOLLOWUP_KEYWORDS = [
    "also", "what about", "how about", "you mentioned", "you said",
    "that", "those", "it", "them", "this", "he", "she", "they",
    "btw", "by the way", "and", "in addition", "one more thing",
    "can you explain", "could you clarify", "tell me more", "give me more details",
    "why is that", "then", "next", "continue", "okay, but", "so...",
    "based on that", "regarding that", "same as earlier", "as I asked",
    "right?", "correct?", "really?"
]

def is_followup_question(message: str, history: list) -> bool:
    if not history:
        return False
    message_lower = message.lower()
    return any(keyword in message_lower for keyword in FOLLOWUP_KEYWORDS)
