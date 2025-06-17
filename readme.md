import requests
import json

# === CONFIGURATION ===
CHAT_API_URL = "http://localhost:8000/v1/chat/completions"  # replace with your endpoint
HEADERS = {"Content-Type": "application/json"}  # add auth headers if needed
TEMPERATURE = 0.2

# === FUNCTION TO SEMANTICALLY CHUNK DOCUMENT ===
def semantic_chunking(document_text: str):
    system_prompt = (
        "You are an AI agent specialized in document analysis. "
        "Your task is to split the given document into meaningful, semantically coherent chunks. "
        "Each chunk should represent a distinct section or topic (e.g., Introduction, Procedure Steps, Summary). "
        "Label each chunk and include the original text under that label."
    )

    user_prompt = (
        f"Split the following document into logical chunks:\n\n{document_text}\n\n"
        "Return the result as a JSON list of objects with keys: 'title' and 'content'."
    )

    payload = {
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        "temperature": TEMPERATURE
    }

    response = requests.post(CHAT_API_URL, headers=HEADERS, data=json.dumps(payload))
    
    if response.status_code != 200:
        raise Exception(f"API Error: {response.status_code}, {response.text}")
    
    response_data = response.json()
    content = response_data["choices"][0]["message"]["content"]

    # Try to parse the returned content as JSON
    try:
        chunks = json.loads(content)
        return chunks
    except json.JSONDecodeError:
        print("Model did not return valid JSON. Here's the raw content:\n", content)
        return None

# === EXAMPLE USAGE ===
if __name__ == "__main__":
    with open("your_document.txt", "r", encoding="utf-8") as f:
        full_text = f.read()

    semantic_chunks = semantic_chunking(full_text)

    if semantic_chunks:
        for idx, chunk in enumerate(semantic_chunks):
            print(f"\n--- Chunk {idx+1}: {chunk['title']} ---")
            print(chunk["content"])

#########
import json
import re
from typing import List, Dict, Optional, Tuple
import requests
from dataclasses import dataclass
from enum import Enum


class ChunkType(Enum):
    """Types of content chunks identified by AI."""
    PROCEDURE = "procedure"
    EXPLANATION = "explanation"
    EXAMPLE = "example"
    DEFINITION = "definition"
    LIST = "list"
    NARRATIVE = "narrative"
    MIXED = "mixed"
    FRAGMENT = "fragment"


@dataclass
class DocumentChunk:
    """Represents a semantically coherent chunk of text."""
    content: str
    topic: str
    start_position: int
    end_position: int
    chunk_type: ChunkType
    confidence: float  # AI confidence in the chunking decision
    semantic_keywords: List[str]  # Key terms that define this chunk


class UnstructuredSemanticChunker:
    """AI-powered semantic chunker optimized for unstructured, messy documents."""
    
    def __init__(self, api_endpoint: str, api_key: str, model: str = "gpt-3.5-turbo"):
        self.api_endpoint = api_endpoint
        self.api_key = api_key
        self.model = model
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }
    
    def _call_api(self, messages: List[Dict], max_tokens: int = 1000) -> str:
        """Make a call to the ChatCompletions API."""
        payload = {
            "model": self.model,
            "messages": messages,
            "max_tokens": max_tokens,
            "temperature": 0.2
        }
        
        response = requests.post(self.api_endpoint, headers=self.headers, json=payload)
        response.raise_for_status()
        
        return response.json()["choices"][0]["message"]["content"]
    
    def _sliding_window_analysis(self, text: str, window_size: int = 800) -> List[Dict]:
        """
        Analyze document using sliding windows to handle unstructured content.
        This works better for documents without clear sections.
        """
        windows = []
        step_size = window_size // 2  # 50% overlap
        
        for i in range(0, len(text), step_size):
            window_text = text[i:i + window_size]
            if len(window_text.strip()) < 100:  # Skip tiny windows
                continue
                
            window_info = {
                "text": window_text,
                "start": i,
                "end": min(i + window_size, len(text))
            }
            windows.append(window_info)
        
        return windows
    
    def _analyze_content_segments(self, text: str) -> List[Dict]:
        """
        Use AI to analyze content and identify semantic segments in unstructured text.
        """
        prompt = f"""
        Analyze this potentially unstructured text and identify coherent content segments.
        The text may contain random steps, explanations, examples mixed together without clear organization.
        
        For each coherent segment you identify, provide:
        1. start_char: approximate starting character position
        2. end_char: approximate ending character position  
        3. content_type: one of [procedure, explanation, example, definition, list, narrative, mixed, fragment]
        4. topic: brief description of what this segment covers
        5. confidence: your confidence (0.0-1.0) that this is a good semantic boundary
        6. keywords: 3-5 key terms that represent this segment
        
        Focus on:
        - Procedural steps (even if scattered)
        - Conceptual explanations 
        - Examples or code snippets
        - Lists or enumerations
        - Definitional content
        - Topic shifts or context changes
        
        Return as JSON array:
        [
            {{
                "start_char": <number>,
                "end_char": <number>,
                "content_type": "<type>",
                "topic": "<brief_topic>",
                "confidence": <0.0-1.0>,
                "keywords": ["keyword1", "keyword2", "keyword3"]
            }},
            ...
        ]
        
        Text to analyze:
        {text[:3000]}{"..." if len(text) > 3000 else ""}
        """
        
        messages = [
            {
                "role": "system", 
                "content": "You are an expert at analyzing unstructured documents and identifying semantic content boundaries. Focus on content coherence rather than formatting."
            },
            {"role": "user", "content": prompt}
        ]
        
        try:
            response = self._call_api(messages, max_tokens=1500)
            json_match = re.search(r'\[.*\]', response, re.DOTALL)
            if json_match:
                segments = json.loads(json_match.group())
                return segments
            return []
        except Exception as e:
            print(f"Error in AI analysis: {e}")
            return []
    
    def _merge_overlapping_segments(self, segments: List[Dict]) -> List[Dict]:
        """
        Merge segments that have significant overlap or are very similar.
        """
        if not segments:
            return []
        
        # Sort segments by start position
        segments = sorted(segments, key=lambda x: x["start_char"])
        merged = [segments[0]]
        
        for current in segments[1:]:
            last = merged[-1]
            
            # Check for overlap
            overlap = max(0, min(last["end_char"], current["end_char"]) - max(last["start_char"], current["start_char"]))
            last_length = last["end_char"] - last["start_char"]
            current_length = current["end_char"] - current["start_char"]
            
            # If significant overlap (>30%) or very similar topics, merge
            if (overlap > 0.3 * min(last_length, current_length) or 
                self._topics_similar(last["topic"], current["topic"])):
                
                # Merge segments
                merged[-1] = {
                    "start_char": last["start_char"],
                    "end_char": max(last["end_char"], current["end_char"]),
                    "content_type": last["content_type"] if last["confidence"] > current["confidence"] else current["content_type"],
                    "topic": last["topic"] if len(last["topic"]) > len(current["topic"]) else current["topic"],
                    "confidence": max(last["confidence"], current["confidence"]),
                    "keywords": list(set(last["keywords"] + current["keywords"]))
                }
            else:
                merged.append(current)
        
        return merged
    
    def _topics_similar(self, topic1: str, topic2: str) -> bool:
        """Check if two topics are similar enough to merge."""
        words1 = set(topic1.lower().split())
        words2 = set(topic2.lower().split())
        
        if not words1 or not words2:
            return False
        
        intersection = words1.intersection(words2)
        union = words1.union(words2)
        
        # Jaccard similarity > 0.3
        return len(intersection) / len(union) > 0.3
    
    def _fill_gaps_between_segments(self, text: str, segments: List[Dict]) -> List[Dict]:
        """
        Identify gaps between segments and decide how to handle them.
        """
        if not segments:
            return []
        
        filled_segments = []
        last_end = 0
        
        for segment in segments:
            start = segment["start_char"]
            
            # If there's a gap, analyze what to do with it
            if start > last_end + 50:  # Significant gap
                gap_text = text[last_end:start].strip()
                if len(gap_text) > 50:  # Substantial gap content
                    # Decide whether to merge with previous, next, or create new segment
                    gap_segment = self._analyze_gap(gap_text, last_end, start, filled_segments, segment)
                    if gap_segment:
                        filled_segments.append(gap_segment)
            
            filled_segments.append(segment)
            last_end = segment["end_char"]
        
        # Handle any remaining text after the last segment
        if last_end < len(text) - 50:
            remaining_text = text[last_end:].strip()
            if len(remaining_text) > 50:
                remaining_segment = {
                    "start_char": last_end,
                    "end_char": len(text),
                    "content_type": "fragment",
                    "topic": "Additional content",
                    "confidence": 0.5,
                    "keywords": self._extract_keywords(remaining_text)
                }
                filled_segments.append(remaining_segment)
        
        return filled_segments
    
    def _analyze_gap(self, gap_text: str, start: int, end: int, prev_segments: List[Dict], next_segment: Dict) -> Optional[Dict]:
        """Analyze gap content and decide if it should be a separate segment."""
        # Simple heuristics for gap analysis
        if len(gap_text) < 100:
            return None  # Too small to be meaningful
        
        # Check if it looks like a continuation of previous content
        if prev_segments:
            prev_keywords = prev_segments[-1]["keywords"]
            gap_keywords = self._extract_keywords(gap_text)
            
            if len(set(prev_keywords).intersection(set(gap_keywords))) >= 2:
                # Likely continuation, merge with previous
                return None
        
        # Create new segment for substantial gaps
        return {
            "start_char": start,
            "end_char": end,
            "content_type": "mixed",
            "topic": "Transitional content",
            "confidence": 0.6,
            "keywords": self._extract_keywords(gap_text)
        }
    
    def _extract_keywords(self, text: str) -> List[str]:
        """Extract key terms from text using simple heuristics."""
        # Remove common words and extract meaningful terms
        words = re.findall(r'\b[a-zA-Z]{3,}\b', text.lower())
        common_words = {'the', 'and', 'for', 'are', 'but', 'not', 'you', 'all', 'can', 'had', 'her', 'was', 'one', 'our', 'out', 'day', 'get', 'has', 'him', 'his', 'how', 'its', 'may', 'new', 'now', 'old', 'see', 'two', 'way', 'who', 'boy', 'did', 'has', 'let', 'put', 'say', 'she', 'too', 'use'}
        
        filtered_words = [w for w in words if w not in common_words and len(w) > 3]
        
        # Get most frequent words
        from collections import Counter
        word_counts = Counter(filtered_words)
        return [word for word, count in word_counts.most_common(5)]
    
    def _create_chunks_from_segments(self, text: str, segments: List[Dict]) -> List[DocumentChunk]:
        """Convert analyzed segments into DocumentChunk objects."""
        chunks = []
        
        for segment in segments:
            start = max(0, segment["start_char"])
            end = min(len(text), segment["end_char"])
            
            content = text[start:end].strip()
            if len(content) < 30:  # Skip very small chunks
                continue
            
            chunk_type = ChunkType(segment["content_type"])
            
            chunk = DocumentChunk(
                content=content,
                topic=segment["topic"],
                start_position=start,
                end_position=end,
                chunk_type=chunk_type,
                confidence=segment["confidence"],
                semantic_keywords=segment["keywords"]
            )
            
            chunks.append(chunk)
        
        return chunks
    
    def _handle_very_large_chunks(self, chunks: List[DocumentChunk], max_size: int = 1500) -> List[DocumentChunk]:
        """Split chunks that are too large while preserving semantic meaning."""
        result = []
        
        for chunk in chunks:
            if len(chunk.content) <= max_size:
                result.append(chunk)
            else:
                # For large chunks, try to split on natural boundaries
                sub_chunks = self._intelligent_split(chunk, max_size)
                result.extend(sub_chunks)
        
        return result
    
    def _intelligent_split(self, chunk: DocumentChunk, max_size: int) -> List[DocumentChunk]:
        """Intelligently split large chunks based on content patterns."""
        content = chunk.content
        
        # Try different splitting strategies based on chunk type
        if chunk.chunk_type == ChunkType.PROCEDURE:
            return self._split_procedure(chunk, max_size)
        elif chunk.chunk_type == ChunkType.LIST:
            return self._split_list(chunk, max_size)
        else:
            return self._split_by_sentences(chunk, max_size)
    
    def _split_procedure(self, chunk: DocumentChunk, max_size: int) -> List[DocumentChunk]:
        """Split procedural content by steps."""
        content = chunk.content
        
        # Look for step indicators
        step_patterns = [
            r'\n\s*\d+[\.\)]\s*',  # 1. or 1)
            r'\n\s*Step\s*\d+',    # Step 1
            r'\n\s*[•\-\*]\s*',    # Bullet points
        ]
        
        splits = []
        for pattern in step_patterns:
            matches = list(re.finditer(pattern, content, re.IGNORECASE))
            if matches:
                splits = [m.start() for m in matches]
                break
        
        if not splits:
            return self._split_by_sentences(chunk, max_size)
        
        # Create sub-chunks based on steps
        sub_chunks = []
        start_pos = 0
        
        for split_pos in splits + [len(content)]:
            if split_pos - start_pos > 50:  # Meaningful chunk size
                sub_content = content[start_pos:split_pos].strip()
                if len(sub_content) > max_size:
                    # Still too large, split by sentences
                    sentence_chunks = self._split_by_sentences(
                        DocumentChunk(sub_content, chunk.topic, 0, len(sub_content), 
                                    chunk.chunk_type, chunk.confidence, chunk.semantic_keywords),
                        max_size
                    )
                    sub_chunks.extend(sentence_chunks)
                else:
                    sub_chunk = DocumentChunk(
                        content=sub_content,
                        topic=chunk.topic,
                        start_position=chunk.start_position + start_pos,
                        end_position=chunk.start_position + split_pos,
                        chunk_type=chunk.chunk_type,
                        confidence=chunk.confidence,
                        semantic_keywords=chunk.semantic_keywords
                    )
                    sub_chunks.append(sub_chunk)
            start_pos = split_pos
        
        return sub_chunks if sub_chunks else [chunk]
    
    def _split_list(self, chunk: DocumentChunk, max_size: int) -> List[DocumentChunk]:
        """Split list content by items."""
        content = chunk.content
        
        # Look for list item patterns
        list_patterns = [
            r'\n\s*[•\-\*]\s*',     # Bullet points
            r'\n\s*\d+[\.\)]\s*',   # Numbered items
            r'\n\s*[a-zA-Z][\.\)]\s*'  # Lettered items
        ]
        
        splits = []
        for pattern in list_patterns:
            matches = list(re.finditer(pattern, content))
            if matches:
                splits = [m.start() for m in matches]
                break
        
        if not splits:
            return self._split_by_sentences(chunk, max_size)
        
        # Group list items to fit within max_size
        sub_chunks = []
        current_content = ""
        current_start = 0
        
        for i, split_pos in enumerate(splits + [len(content)]):
            item_content = content[current_start if i == 0 else splits[i-1]:split_pos]
            
            if len(current_content + item_content) > max_size and current_content:
                # Create chunk with current items
                sub_chunk = DocumentChunk(
                    content=current_content.strip(),
                    topic=chunk.topic,
                    start_position=chunk.start_position + current_start,
                    end_position=chunk.start_position + (splits[i-1] if i > 0 else split_pos),
                    chunk_type=chunk.chunk_type,
                    confidence=chunk.confidence,
                    semantic_keywords=chunk.semantic_keywords
                )
                sub_chunks.append(sub_chunk)
                current_content = item_content
                current_start = splits[i-1] if i > 0 else 0
            else:
                current_content += item_content
        
        # Add remaining content
        if current_content.strip():
            sub_chunk = DocumentChunk(
                content=current_content.strip(),
                topic=chunk.topic,
                start_position=chunk.start_position + current_start,
                end_position=chunk.end_position,
                chunk_type=chunk.chunk_type,
                confidence=chunk.confidence,
                semantic_keywords=chunk.semantic_keywords
            )
            sub_chunks.append(sub_chunk)
        
        return sub_chunks if sub_chunks else [chunk]
    
    def _split_by_sentences(self, chunk: DocumentChunk, max_size: int) -> List[DocumentChunk]:
        """Split chunk by sentences as fallback method."""
        content = chunk.content
        sentences = re.split(r'(?<=[.!?])\s+', content)
        
        sub_chunks = []
        current_content = ""
        current_start = 0
        current_pos = 0
        
        for sentence in sentences:
            if len(current_content + sentence) > max_size and current_content:
                sub_chunk = DocumentChunk(
                    content=current_content.strip(),
                    topic=chunk.topic,
                    start_position=chunk.start_position + current_start,
                    end_position=chunk.start_position + current_pos,
                    chunk_type=chunk.chunk_type,
                    confidence=chunk.confidence * 0.8,  # Lower confidence for forced splits
                    semantic_keywords=chunk.semantic_keywords
                )
                sub_chunks.append(sub_chunk)
                current_content = sentence
                current_start = current_pos
            else:
                current_content += " " + sentence if current_content else sentence
            current_pos += len(sentence) + 1
        
        # Add remaining content
        if current_content.strip():
            sub_chunk = DocumentChunk(
                content=current_content.strip(),
                topic=chunk.topic,
                start_position=chunk.start_position + current_start,
                end_position=chunk.end_position,
                chunk_type=chunk.chunk_type,
                confidence=chunk.confidence * 0.8,
                semantic_keywords=chunk.semantic_keywords
            )
            sub_chunks.append(sub_chunk)
        
        return sub_chunks if sub_chunks else [chunk]
    
    def chunk_document(self, text: str, max_chunk_size: int = 1500, min_confidence: float = 0.3) -> List[DocumentChunk]:
        """
        Main method to chunk unstructured documents semantically.
        
        Args:
            text: The document text to chunk
            max_chunk_size: Maximum size for chunks
            min_confidence: Minimum confidence threshold for chunks
            
        Returns:
            List of DocumentChunk objects
        """
        print("Analyzing unstructured document...")
        
        # For very long documents, process in overlapping windows
        if len(text) > 4000:
            print("Large document detected, using windowed analysis...")
            return self._process_large_document(text, max_chunk_size, min_confidence)
        
        
