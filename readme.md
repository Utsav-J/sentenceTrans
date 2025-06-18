import openai
import uuid

def semantic_chunking(document_text: str, model: str = "gpt-4o") -> dict:
    prompt = f"""
You are an AI document processor. Your job is to semantically chunk the document into meaningful sections.

**Goal**:
Split the document into sections such as:
- Step-by-step procedures (MUST be in a separate chunk if present)
- Information retrieval-related sections
- Summaries
- Any other semantically unique sections

Return the result in the following JSON format:
```json
{{
  "documentTitle": "Title of the Document",
  "semanticChunks": [
    {{
      "chunkId": "uuid-generated-chunk-id",
      "chunkType": "step_by_step_procedure",
      "chunkTitle": "How to Configure XYZ",
      "content": "The detailed step-by-step instructions..."
    }},
    {{
      "chunkId": "uuid-generated-chunk-id",
      "chunkType": "information_retrieval",
      "chunkTitle": "Overview of IR Techniques",
      "content": "This section talks about..."
    }}
  ]
}}
