# Wipro — Generative AI Engineer Technical Interview Guide (Coding & Production Systems Round)

This document provides a comprehensive, production-grade guide on a **Generative AI Engineer Coding & System Resilience Round** at **Wipro**. 

Unlike standard algorithmic coding sessions, this interview evaluates how candidates translate real-world LLM production failures (flaky APIs, high latency, context limits, retrieval noise, structured output validation) into resilient Python code and backend system designs.

---

## 1. Candidate Background & Interview Overview

* **Company & Role:** Generative AI Engineer / LLM Engineer at **Wipro** (Walk-in Drive & Technical Rounds).
* **Format:** Interactive Hands-on Live Coding & Applied Production System Design.
* **Evaluation Core:** Rather than static LeetCode-style data structures, the interviewer constructs dynamic, evolving production edge cases around candidate code solutions (e.g., *"What if the API returns 400?"*, *"What if cached data goes stale?"*, *"What if all top retrieved documents are duplicates?"*).
* **Key Technical Domains Covered:**
  1. Resilient External API Execution (Exponential Backoff, Jitter, Status Handling).
  2. Cost Optimization & Latency Control via Caching (In-Memory, Redis, TTL Strategies).
  3. Context Window Management & Chunking Strategies (Fixed-size, Overlap, Semantic Chunking).
  4. Advanced RAG Retrieval & Diversity Re-ranking (Top-K, Over-retrieval, Cross-Encoders).
  5. Output Validation & Guardrails for Dynamic LLM Schemas (Pydantic, Structural Fallbacks).

---

## 2. Dynamic Interview Questions, Code Implementations & Production Edge Cases

### Problem 1: Flaky External LLM APIs & Resilient Retries

#### Base Question:
How do you write a Python wrapper/decorator to handle intermittent failures when calling external LLM APIs with exponential backoff?

#### Candidate Approach & Code Solution:
Implement an exponential backoff loop bounded by a maximum retry count to prevent infinite execution loops in production environments.

```python
import time
import random
import logging
from typing import Callable, Any

logging.basicConfig(level=logging.INFO)

def call_llm_api_stub() -> str:
    """Simulated LLM API call that may fail intermittently."""
    if random.random() < 0.7:
        raise TimeoutError("504 Gateway Timeout: LLM provider unresponsive")
    return "LLM Response: Generation Successful"

def execute_with_exponential_backoff(
    api_func: Callable[[], Any],
    max_retries: int = 3,
    base_delay: float = 1.0,
    backoff_factor: float = 2.0
) -> Any:
    """
    Executes an API call with configurable retry counts and exponential delays.
    """
    delay = base_delay
    for attempt in range(1, max_retries + 1):
        try:
            logging.info(f"Attempt {attempt}/{max_retries}: Calling LLM API...")
            return api_func()
        except Exception as err:
            logging.warning(f"Attempt {attempt} failed: {err}")
            if attempt == max_retries:
                logging.error("Max retries reached. Escalating exception.")
                raise err
            
            time.sleep(delay)
            delay *= backoff_factor

```

---

#### Follow-Up Questions & Production Hardening

##### 1. Interviewer Question: *"What if the API returns a `400 Bad Request` or `401 Unauthorized`? Will your function retry?"*

* **Candidate Answer:** No, non-transient client errors ($4xx$) should **never** be retried. A `400` indicates a malformed payload or invalid prompt parameters, and a `401` indicates authentication failure. Retrying these will yield identical failures, waste compute, and exhaust rate limits. Retries should strictly target **transient network faults** ($502, 503, 504$) and **rate limits** ($429$).

##### 2. Interviewer Question: *"How do you make this retry pattern resilient when scaling to thousands of concurrent users?"*

* **Candidate Answer:** Apply **Full Jitter** to the exponential delay. If thousands of worker threads encounter a micro-outage simultaneously, synchronized retries cause a "Thundering Herd" problem, repeatedly overloading the downstream API server at exact interval spikes. Jitter spreads out retry traffic across randomized time windows.

##### Production-Grade Retry Implementation (With Status Codes & Jitter):

```python
import time
import random
import logging
from typing import Callable, Any

class HTTPClientError(Exception):
    def __init__(self, status_code: int, message: str):
        self.status_code = status_code
        super().__init__(f"HTTP {status_code}: {message}")

class HTTPServerError(Exception):
    def __init__(self, status_code: int, message: str):
        self.status_code = status_code
        super().__init__(f"HTTP {status_code}: {message}")

def execute_production_resilient_retry(
    api_func: Callable[[], Any],
    max_retries: int = 4,
    base_delay: float = 1.0,
    max_delay: float = 16.0
) -> Any:
    """
    Production retry pattern with 4xx filtering, exponential backoff, and full jitter.
    """
    for attempt in range(1, max_retries + 1):
        try:
            return api_func()
        except HTTPClientError as e:
            # 4xx errors are client-side errors (bad params, auth issues); terminate retries immediately
            logging.error(f"[NON-RETRYABLE] Client error encounterd: {e}")
            raise e
        except (HTTPServerError, TimeoutError, ConnectionError) as e:
            # 5xx errors and network drops are transient; proceed with retries
            if attempt == max_retries:
                logging.error(f"[FAILED] Exhausted all {max_retries} retry attempts.")
                raise e
            
            # Calculate Exponential Backoff with Full Jitter
            calculated_delay = min(max_delay, base_delay * (2 ** (attempt - 1)))
            jittered_delay = random.uniform(0, calculated_delay)
            
            logging.warning(
                f"[RETRY {attempt}/{max_retries}] Transient failure ({e}). "
                f"Retrying in {jittered_delay:.2f} seconds..."
            )
            time.sleep(jittered_delay)

```

---

### Problem 2: Response Caching, Cost Optimization & Cache Invalidation

#### Base Question:

Frequent identical user queries significantly inflate LLM API billing and increase latency. How do you implement a caching system to mitigate this?

#### Candidate Approach & Code Solution:

Implement a query-level key-value lookup. Hashing the normalized prompt string generates a cache key to return pre-computed responses instantly.

```python
import hashlib
from typing import Dict, Optional

class SimpleLLMCache:
    def __init__(self):
        self._cache: Dict[str, str] = {}

    def _generate_key(self, query: str) -> str:
        # Normalize whitespace and lower string before hashing
        normalized_query = " ".join(query.strip().lower().split())
        return hashlib.sha256(normalized_query.encode('utf-8')).hexdigest()

    def get(self, query: str) -> Optional[str]:
        key = self._generate_key(query)
        return self._cache.get(key)

    def set(self, query: str, response: str) -> None:
        key = self._generate_key(query)
        self._cache[key] = response

```

---

#### Follow-Up Questions & Production Hardening

##### 1. Interviewer Question: *"What are the key limitations of relying on an in-memory dictionary cache in production?"*

* **Candidate Answer:**
* **Memory Volatility:** In-memory caches clear when the service container restarts or crashes.
* **Instance Isolation:** Modern applications deploy across multiple stateless server instances (e.g., Kubernetes pods). In-memory dicts create isolated, inconsistent local caches, resulting in duplicate LLM calls across pods.
* **Fix:** Use a centralized distributed cache like **Redis**, paired with an LRU (Least Recently Used) eviction policy to control memory footprint.



##### 2. Interviewer Question: *"Would you blindly cache every LLM response?"*

* **Candidate Answer:** No. Caching non-deterministic or real-time context queries (e.g., *"What is the current stock price of Apple?"*, *"Generate a personalized greeting for User X"*) causes stale, invalid outputs.

##### Caching Invalidation Strategy Matrix:

| Query Type | Caching Decision | Invalidation / TTL Strategy |
| --- | --- | --- |
| **Static Knowledge** (e.g., *"Explain backpropagation"*) | **Cache** | Long TTL (e.g., 30 Days) |
| **Semi-Static Docs** (e.g., Enterprise HR policies) | **Cache** | Medium TTL (e.g., 24 Hours) + Event-Driven Invalidation when document updates |
| **Real-time / Dynamic Context** (e.g., Live feeds, stock prices) | **Do Not Cache** | TTL = `0` (Bypass Cache entirely) |
| **Personalized User Session** | **Cache with Session Scope** | Key composed of `hash(user_id + prompt)` + Short TTL (e.g., 15 mins) |

---

### Problem 3: Context Window Constraints & Document Chunking

#### Base Question:

Large documents often exceed the context window of an LLM. How do you split raw text into chunks, and why is chunk overlap necessary?

#### Candidate Approach & Code Solution:

Implement a sliding-window text chunker using configurable character limits and overlap intervals to maintain continuity across chunk boundaries.

```python
from typing import List

def chunk_text_with_overlap(text: str, chunk_size: int, chunk_overlap: int) -> List[str]:
    """
    Splits text into overlapping chunks using character-level sliding windows.
    """
    if chunk_overlap >= chunk_size:
        raise ValueError("chunk_overlap must be strictly less than chunk_size")
    
    chunks = []
    start = 0
    text_length = len(text)

    while start < text_length:
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        
        # Advance the window by stride (size - overlap)
        start += (chunk_size - chunk_overlap)
        
    return chunks

# Test Execution
sample_doc = "Generative AI systems require scalable data pipelines and effective chunking strategies."
chunks = chunk_text_with_overlap(sample_doc, chunk_size=30, chunk_overlap=10)

```

---

#### Follow-Up Questions & Production Hardening

##### 1. Interviewer Question: *"Why is chunk overlap critical for retrieval accuracy?"*

* **Candidate Answer:** Text chunking creates artificial boundaries. If a critical semantic fact spans across the boundary of Chunk A and Chunk B (e.g., a entity subject in Chunk A and its primary action verb in Chunk B), splitting without overlap fragments the context. Embedding models will lose the core context of both fragments. Overlap preserves semantic continuity across contiguous chunks.

##### 2. Interviewer Question: *"Is fixed-character chunking optimal for all document types?"*

* **Candidate Answer:** No. Fixed-character splitting ignores semantic structure, often cutting sentences or code blocks mid-word.
* **Recursive Character Splitting:** Hierarchy-aware splitting that attempts to preserve structural boundaries by splitting sequentially on `\n\n` (paragraphs), `\n` (lines), ` ` (words), and finally `""` (characters).
* **Semantic Chunking:** Computes similarity between adjacent sentences using embeddings, placing split points only where semantic distance spikes significantly.



```python
import re
from typing import List

def recursive_character_tokenizer_concept(text: str, max_chunk_size: int) -> List[str]:
    """
    Splits content primarily on paragraph delimiters, falling back to lower-level
    separators only when paragraph blocks exceed max_chunk_size.
    """
    paragraphs = text.split("\n\n")
    final_chunks = []
    current_chunk = []
    current_length = 0

    for para in paragraphs:
        if current_length + len(para) <= max_chunk_size:
            current_chunk.append(para)
            current_length += len(para) + 2  # account for delimiter
        else:
            if current_chunk:
                final_chunks.append("\n\n".join(current_chunk))
            current_chunk = [para]
            current_length = len(para)

    if current_chunk:
        final_chunks.append("\n\n".join(current_chunk))

    return final_chunks

```

---

### Problem 4: Retrieval Diversity & Re-Ranking Pipeline

#### Base Question:

A vector database returns the top 20 candidate documents based on cosine similarity, but you must pass only the 5 most relevant chunks to the LLM context. Write a function to extract top results.

#### Candidate Code Solution (Initial Sorting):

```python
from typing import List, Dict, Any

def get_top_k_documents(retrieved_docs: List[Dict[str, Any]], top_k: int = 5) -> List[Dict[str, Any]]:
    """
    Sorts retrieved vector search results by similarity score and returns top_k.
    """
    sorted_docs = sorted(retrieved_docs, key=lambda x: x.get("score", 0.0), reverse=True)
    return sorted_docs[:top_k]

```

---

#### Follow-Up Questions & Production Hardening

##### 1. Interviewer Question: *"What if the top 5 retrieved documents are near-identical copies of each other?"*

* **Candidate Answer:** High vector similarity does not guarantee information diversity. If a database contains repeated policy updates or redundant log entries, standard cosine sorting returns $5$ near-duplicate chunks. This consumes valuable context window space without adding useful information.

##### 2. Interviewer Question: *"How do you solve this near-duplicate retrieval issue in production?"*

* **Candidate Answer:** Implement a **Two-Stage Retrieval Architecture** utilizing **Over-Retrieval + Re-Ranking**.
1. **Stage 1 (High Recall):** Retrieve an expanded candidate set (e.g., Top $K=30$) from the vector store using fast approximate nearest neighbors (ANN).
2. **Stage 2 (High Precision & Diversity):** Pass candidates through a **Cross-Encoder Re-Ranker** (e.g., `Cohere Rerank` or `bge-reranker-large`) or run Maximal Marginal Relevance (MMR) to maximize information coverage while penalizing redundant text embeddings.



```
                    [ User Query ]
                          │
                          ▼
             [ Stage 1: Vector Search ]
              (Fast ANN, High Recall)
                          │
                          ▼
            [ Top 30 Candidate Documents ]
                          │
                          ▼
              [ Stage 2: Cross-Encoder ]
            (Contextual Re-Ranking & MMR)
                          │
                          ▼
         [ Top 5 Diverse & Relevant Chunks ]
                          │
                          ▼
                     [ LLM Context ]

```

##### Re-Ranking & Diversity Implementation Concept:

```python
from typing import List, Dict, Any

def mock_cross_encoder_rerank(query: str, documents: List[str]) -> List[float]:
    """Simulates a Cross-Encoder scoring relevant context directly against query."""
    # Cross-encoders evaluate Query + Document jointly, producing accurate relevance scores
    return [round(0.95 - (i * 0.05), 2) for i in range(len(documents))]

def execute_two_stage_retrieval(
    query: str, 
    raw_vector_candidates: List[Dict[str, Any]], 
    final_top_k: int = 5
) -> List[Dict[str, Any]]:
    """
    Over-retrieves candidate chunks, scores them via re-ranker, and returns top diversity set.
    """
    # 1. Extract text payloads from over-retrieved vector store candidates
    doc_texts = [doc["text"] for doc in raw_vector_candidates]
    
    # 2. Compute true contextual relevance scores via re-ranker
    rerank_scores = mock_cross_encoder_rerank(query, doc_texts)
    
    # 3. Attach new scores to document objects
    for idx, doc in enumerate(raw_vector_candidates):
        doc["rerank_score"] = rerank_scores[idx]
        
    # 4. Sort strictly by contextual re-ranker output
    reranked_docs = sorted(raw_vector_candidates, key=lambda x: x["rerank_score"], reverse=True)
    
    return reranked_docs[:final_top_k]

```

---

### Problem 5: LLM Structured Output Parsing, Validation & Guardrails

#### Base Question:

An downstream service expects the LLM to return valid JSON, but the model occasionally returns malformed JSON, markdown formatting (````json ... ````), or conversational prefix/suffix text. How do you handle this?

#### Candidate Approach & Code Solution:

Extract the raw JSON string using regex patterns, parse it safely, and validate it against an expected model structure.

```python
import re
import json
from typing import Dict, Any

def sanitize_and_parse_json(raw_llm_output: str) -> Dict[str, Any]:
    """
    Strips conversational text and markdown blocks to parse underlying JSON safely.
    """
    # Pattern to extract content enclosed inside markdown json blocks or raw braces
    json_pattern = r"```(?:json)?\s*(\{.*?\})\s*```|(\{.*\})"
    match = re.search(json_pattern, raw_llm_output, re.DOTALL)
    
    if match:
        clean_json_str = match.group(1) or match.group(2)
    else:
        clean_json_str = raw_llm_output.strip()
        
    try:
        return json.loads(clean_json_str)
    except json.JSONDecodeError as err:
        raise ValueError(f"Failed to parse extracted text into valid JSON: {err}")

```

---

#### Follow-Up Questions & Production Hardening

##### 1. Interviewer Question: *"What if the model repeatedly generates invalid JSON payloads even after retrying?"*

* **Candidate Answer:**
1. **Schema Enforcement:** Use **Pydantic** to enforce strict structural and data-type validation beyond basic JSON syntax checks.
2. **Self-Correction Retry Loop:** Pass the malformed generation and the specific validation error message back to the LLM in a targeted follow-up prompt, requesting a corrected payload.
3. **Graceful Fallbacks:** If the model fails after $N$ retry attempts, log the payload for monitoring, fire an alert, and trigger a structured fallback response to maintain upstream service stability.



##### Complete Structured Guardrail Implementation with Pydantic Validation:

```python
from pydantic import BaseModel, Field, ValidationError
import json
import logging

# Define Expected Pydantic Output Schema
class UserSentimentResponse(BaseModel):
    sentiment: str = Field(description="Must be 'Positive', 'Negative', or 'Neutral'")
    confidence_score: float = Field(ge=0.0, le=1.0)
    key_topics: list[str]

def validate_llm_response_schema(raw_response: str) -> UserSentimentResponse:
    """
    Parses and validates LLM generation against a deterministic Pydantic schema.
    """
    # Parse raw string output to dictionary
    parsed_json = sanitize_and_parse_json(raw_response)
    
    # Enforce Pydantic validation rules
    return UserSentimentResponse(**parsed_json)

# Execution Sandbox Demonstrating Validation Fallback Handling
if __name__ == "__main__":
    valid_mock_llm_output = """
    Here is the requested output:
    ```json
    {
        "sentiment": "Positive",
        "confidence_score": 0.94,
        "key_topics": ["latency", "reliability", "python"]
    }
    ```
    Hope this helps!
    """
    
    invalid_mock_llm_output = """
    ```json
    {
        "sentiment": "InvalidSentimentChoice",
        "confidence_score": 1.5,
        "key_topics": "NotAList"
    }
    ```
    """

    # Test 1: Successful Validation
    try:
        validated_data = validate_llm_response_schema(valid_mock_llm_output)
        print(f"[SUCCESS] Validated Structured Output: {validated_data}")
    except Exception as e:
        print(f"[FAILED] {e}")

    # Test 2: Validation Failure Handling
    try:
        validated_data = validate_llm_response_schema(invalid_mock_llm_output)
    except ValidationError as val_err:
        logging.error(f"[SCHEMA VALIDATION ERROR] Output violated bounds:\n{val_err}")
        # Route to Self-Correction Loop or Return Fallback System Payload

```

---

## 3. Executive Checklist for Senior Gen AI Coding Rounds

| Stage | Key Objective | Candidate Action Required |
| --- | --- | --- |
| **Step 1: Clarify Assumptions** | Scope production edge cases before typing code. | Ask: *"What are the scaling constraints? Should I handle network timeouts, rate limits, or non-200 HTTP statuses?"* |
| **Step 2: Verbalize System Trade-offs** | Prevent silent implementation choices. | Explain aloud: *"I am selecting an in-memory dictionary for this prototype, but for multi-instance production, we must swap this with Redis."* |
| **Step 3: Guard Edge Cases** | Write failure-tolerant logic. | Enforce explicit input validation, handle empty vectors, set explicit HTTP timeouts, and filter non-retryable statuses ($4xx$). |
| **Step 4: Bridge Code & Architecture** | Connect raw Python functions to cloud infrastructure. | Link your functions to logging tools, monitoring frameworks, cache invalidation policies, and fallback strategies. |

```

```
