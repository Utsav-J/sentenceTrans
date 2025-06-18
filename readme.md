{
  "type": "object",
  "properties": {
    "documentTitle": { "type": "string" },
    "semanticChunks": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "chunkId": { "type": "string" },
          "chunkType": {
            "type": "string",
            "enum": ["step_by_step_procedure", "information_retrieval", "summary", "other"]
          },
          "chunkTitle": { "type": "string" },
          "content": { "type": "string" }
        },
        "required": ["chunkId", "chunkType", "chunkTitle", "content"]
      }
    }
  },
  "required": ["documentTitle", "semanticChunks"]
}
