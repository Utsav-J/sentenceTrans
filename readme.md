import copy
from datetime import datetime

def enrich_chunks(
    json_data: dict,
    file_name: str,
    document_id: str,
    usecase_id: str = "usecase",
    fallback_date: str = None
) -> dict:
    if fallback_date is None:
        fallback_date = datetime.utcnow().isoformat() + "Z"  # ISO format fallback timestamp

    enriched_chunks = []
    for idx, chunk in enumerate(json_data.get("semanticChunks", []), start=1):
        enriched_chunk = copy.deepcopy(chunk)  # Avoid modifying original

        enriched_chunk.update({
            "sor_last_modified": fallback_date,
            "usecase_id": usecase_id,
            "document_id": document_id,
            "chunk_id": f"{document_id}_{idx}",
            "data_classification": "internal",
            "raw_context": chunk.get("content", ""),
            "chunk_intent": chunk.get("chunkIntent", ""),  # maps to old key
            "file_name": file_name
        })

        enriched_chunks.append(enriched_chunk)

    # Return the updated document with enriched chunks
    return {
        "documentTitle": json_data.get("documentTitle", ""),
        "documentSummary": json_data.get("documentSummary", ""),
        "semanticChunks": enriched_chunks
    }
