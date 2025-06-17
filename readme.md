import json
from collections import defaultdict
import google.generativeai as genai

genai.configure(api_key="YOUR_API_KEY")
model = genai.GenerativeModel("gemini-1.5-pro")

input_path = "your_chunks.jsonl"
output_path = "tagged_chunks.jsonl"
doc_intent_output = "document_intents.jsonl"

document_chunks = defaultdict(list)
tagged_chunks = []

# 1. Read chunks and assign ChunkIntent
with open(input_path, "r") as f_in, open(output_path, "w") as f_out:
    for line in f_in:
        entry = json.loads(line)
        raw_text = entry["RawContext"]

        # Generate ChunkIntent using Gemini
        prompt = f"""
You are a helpful assistant. Analyze this academic chunk and assign an intent tag.
Suggested tags: definition, process, comparison, example, code, application, limitation, equation, other.

Raw Text: {raw_text}
Only return the tag.
"""
        chat = model.start_chat(history=[])
        response = chat.send_message(prompt)
        intent = response.text.strip().lower()

        # Update entry
        entry["ChunkIntent"] = intent
        tagged_chunks.append(entry)

        # Save to file
        f_out.write(json.dumps(entry) + "\n")

        # Group by DocumentId
        document_chunks[entry["DocumentId"]].append(entry)

# 2. Assign DocumentIntent per Document
with open(doc_intent_output, "w") as doc_out:
    for doc_id, chunks in document_chunks.items():
        chunk_intents = [chunk["ChunkIntent"] for chunk in chunks]
        raw_contexts = [chunk["RawContext"] for chunk in chunks]

        prompt = f"""
You are a document summarization agent. Given the intents of chunks below, classify the overall intent of the document.
Chunk Intents: {chunk_intents}

Optional: Contexts:\n{"\n".join(raw_contexts[:5])}  # limit to avoid token overflow

Return only one tag for the document: definition, comparison, application, tutorial, research_summary, other.
"""
        chat = model.start_chat(history=[])
        response = chat.send_message(prompt)
        doc_intent = response.text.strip().lower()

        doc_out.write(json.dumps({
            "DocumentId": doc_id,
            "DocumentIntent": doc_intent
        }) + "\n")
