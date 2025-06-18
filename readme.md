import openai
import uuid

def merge_and_format_process_guidance_chunks(json_data: dict, model: str = "gpt-4o") -> dict:
    # 1. Filter and combine all process_guidance chunks
    guidance_chunks = [
        chunk["content"] for chunk in json_data.get("semanticChunks", [])
        if chunk.get("chunkType") == "step_by_step_procedure"
    ]

    if not guidance_chunks:
        return {"error": "No process guidance chunks found."}

    combined_content = "\n\n".join(guidance_chunks)

    # 2. Pass combined content through ChatCompletions to clean up formatting
    prompt = f"""
You are a document formatting assistant.

Below is a combined set of step-by-step procedures extracted from a technical document. Please:
- Clean up the formatting
- Preserve the order of steps
- Use numbered or bulleted lists where appropriate
- Add proper headings or subheadings if necessary
- Ensure the final version is suitable for inclusion in a professional technical document.

### Raw Input:
\"\"\"{combined_content}\"\"\"
"""

    response = openai.ChatCompletion.create(
        model=model,
        messages=[
            {"role": "system", "content": "You are an expert technical document formatter."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.3
    )

    formatted_output = response.choices[0].message["content"]

    # 3. Return as a single new chunk
    merged_chunk = {
        "chunkId": str(uuid.uuid4()),
        "chunkType": "step_by_step_procedure",
        "chunkTitle": "Merged and Formatted Step-by-Step Procedure",
        "content": formatted_output
    }

    return merged_chunk
