{
  "type": "object",
  "properties": {
    "process_name": { "type": "string" },
    "steps": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "step_number": { "type": "integer" },
          "step_title": { "type": "string" },
          "description": { "type": "string" },
          "sub_steps": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "sub_step_number": { "type": "integer" },
                "sub_step_title": { "type": "string" },
                "description": { "type": "string" }
              },
              "required": ["sub_step_number", "sub_step_title", "description"]
            }
          }
        },
        "required": ["step_number", "step_title", "description"]
      }
    }
  },
  "required": ["process_name", "steps"]
}

import openai
import json

openai.api_key = "your_openai_api_key"

messages = [
    {
        "role": "system",
        "content": "You are an AI assistant that extracts detailed step-by-step processes from text and outputs them in structured JSON format according to a predefined schema."
    },
    {
        "role": "user",
        "content": "Input: To prepare a cup of tea, first boil water. Then add tea leaves. Let it simmer for a few minutes. Add milk and sugar as per taste. Strain and serve hot.\n\nExtract the step-by-step process from this text and return it as a structured JSON object."
    }
]

response = openai.ChatCompletion.create(
    model="gpt-4",  # or gpt-4o
    messages=messages,
    temperature=0.2,
    response_format="json"  # use this for OpenAI to enforce JSON mode
)

output = response.choices[0].message["content"]
parsed_json = json.loads(output)

print(json.dumps(parsed_json, indent=2))

