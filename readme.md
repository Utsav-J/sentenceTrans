def update_semantic_chunks_with_merged_guidance(json_data: dict, merged_chunk: dict) -> dict:
    # Step 1: Filter out process_guidance chunks
    retained_chunks = [
        chunk for chunk in json_data.get("semanticChunks", [])
        if chunk.get("chunkType") != "step_by_step_procedure"
    ]

    # Step 2: Append the merged guidance chunk
    retained_chunks.append(merged_chunk)

    # Step 3: Reassign sequential chunk IDs starting from 1
    for i, chunk in enumerate(retained_chunks, start=1):
        chunk["chunkId"] = str(i)

    # Step 4: Update the JSON and return
    updated_json = {
        "documentTitle": json_data.get("documentTitle", "Untitled Document"),
        "semanticChunks": retained_chunks
    }

    return updated_json
