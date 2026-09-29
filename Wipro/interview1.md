# Wipro Generative AI Engineer Walk-In Interview Guide

---

## 1. Candidate Background & Interview Process Summary

* **Role & Company:** Generative AI Engineer at Wipro (Walk-In Drive).
* **Application Pathway:** Online job portal application $\rightarrow$ HR phone screening $\rightarrow$ On-site Saturday walk-in interview.
* **Format:**
  * **HR Phone Screening:** Confirmed current role, total years of experience, notice period, expected CTC, and technical stack.
  * **3-Day Sprint Preparation Strategy:**
    * *Day 1:* RAG pipeline, chunking strategies, embeddings, vector databases, and retrieval logic.
    * *Day 2:* LLM fundamentals (Attention mechanisms, Temperature, Top-P, Fine-tuning vs. RAG, Hallucination mitigation).
    * *Day 3:* AI Agents, tool calling/schemas, Model Context Protocol (MCP), evaluation frameworks, and project architecture rehearsal.
  * **Technical Interview (40 Minutes):** Conducted by a Senior Engineer and a Technical Lead. Covered project architecture, advanced retrieval debugging, LLM security (prompt injection), memory management, rapid-fire fundamentals, and live Python coding with system-design explanations.

---

## 2. Advanced RAG, Retrieval Optimization & Debugging

### Question 1: How do you improve document retrieval when user queries diverge significantly from the raw context text?
**Answer:**
* **Query Rewriting / Expansion:** Use a lightweight LLM step to rephrase, expand, or break down messy or ambiguous user queries into well-formed sub-queries before searching the vector index.
* **Hypothetical Document Embeddings (HyDE):** Prompt an LLM to generate a hypothetical answer to the user's question, embed that generated answer, and execute vector similarity search against real context chunks.
* **Hybrid Search (Dense + Sparse):** Combine **Dense Vector Retrieval** (capturing semantic intent via embeddings) with **Sparse Keyword Retrieval** (e.g., BM25 for exact match acronyms, IDs, or medical/legal terminology) using Reciprocal Rank Fusion (RRF).
* **Cross-Encoder Reranking:** Fetch a larger candidate pool (e.g., top-50 chunks) via vector search and pass them through a cross-encoder model (e.g., Cohere Rerank, BGE-Reranker) to output fine-grained relevancy scores for the final top-$K$ selection.

---

### Question 2: If the retrieved chunks are accurate but the LLM still generates an incorrect or incomplete answer, how do you debug the system?
**Answer:**


```

```
                    [ User Query ]
                          │
                          ▼
               [ Vector Index Search ]
                          │
                          ▼
              [ Context Chunks Loaded ]
                          │
         ┌────────────────┴────────────────┐
         ▼                                 ▼

```

[ Context Precision /             [ Context Chunk
Recall Check ]                   Ordering Check ]
(Are key facts missing           (Evaluate "Lost in the
or contaminated?)                  Middle" positioning)
│                                 │
└────────────────┬────────────────┘
│
▼
[ Prompt Template Audit ]
(Check system guardrails, rules,
and delimiter separation)
│
▼
[ Parameter Configuration ]
(Lower Temperature to ~0.0;
adjust Top-P parameters)
│
▼
[ Final Model Isolation ]
(Test with larger LLM context window /
higher-capacity reasoning model)

```

1. **Isolate Retrieval vs. Generation:** Evaluate context precision and recall using metrics (e.g., RAGAS) to confirm whether the necessary facts exist within the retrieved chunks.
2. **Inspect Context Ordering ("Lost in the Middle"):** LLMs tend to pay more attention to tokens at the very beginning and end of long prompts. Reorder top chunks so the highest-scoring passages sit at the top or bottom rather than hidden in the middle.
3. **Audit Prompt Template & Delimiters:** Ensure clear boundary markers (e.g., `<context> ... </context>`) separate retrieved passages from system instructions, preventing the model from confusing context with user rules.
4. **Tune Generation Hyperparameters:** Reduce the model's temperature (e.g., set to $0.0$) and adjust Top-P to enforce deterministic, factual extraction instead of loose creative sampling.

---

## 3. LLM Security, Memory & Operations

### Question 3: What is Prompt Injection, and how do you protect production Gen AI applications against it?
**Answer:**
* **Definition:** Prompt injection occurs when untrusted user input contains malicious instructions designed to bypass system prompts, override safety rules, exfiltrate system instructions, or hijack tool calls.
* **Defense-in-Depth Mitigation Strategy:**
  * **Input Validation & Sanitization:** Strip harmful control tags, enforce strict character/token limits, and run user inputs through dedicated moderation classifiers (e.g., Llama Guard) prior to prompt construction.
  * **Strict Structural Separation:** Keep system instructions and variable user inputs isolated using clear API structures or prompt boundary tags rather than raw string concatenation.
  * **Privilege Restriction on Tools:** Enforce least-privilege access on external function/tool integrations (e.g., restrict database tools to read-only views; mandate human confirmation for side-effects).
  * **Output Guardrails & Filtering:** Inspect generated outputs for sensitive data leaks (PII) or unauthorized system prompt regurgitation before returning responses to users.

---

### Question 4: How do you handle long-term conversation memory in LLMs without exceeding context window limits or inflating token costs?
**Answer:**
* **Sliding Window Buffer:** Retain only the most recent $N$ message turns (e.g., last 5 interactions) while discarding older messages.
* **Conversation Summarization:** Use a background LLM process to periodically condense older conversation turns into a running summary block appended to the system prompt.
* **Vector-Backed Episodic Memory:** Store historic user interactions in a vector database. Retrieve relevant past dialogue turns dynamically using semantic similarity search based on the user's active topic.

---

## 4. Rapid-Fire Technical Fundamentals

* **Embeddings:** High-dimensional vector representations of text that capture semantic meaning, context, and relationships in a continuous vector space.
* **Chunk Overlap:** Including duplicate trailing/leading tokens across adjacent text chunks to prevent losing semantic context split across chunk boundaries.
* **Temperature = 0:** Forces greedy decoding, causing the LLM to pick the highest-probability next token at every step for maximum determinism.
* **Reranker:** A cross-encoder model that scores query-document pairs together to compute an accurate relevancy ranking over a preliminary list of candidates retrieved via vector search.

---

## 5. Live Coding Challenges (with Production Explanations)

### Task 1: Write a Python function that executes an API call with exponential backoff retries when encountering errors.

```python
import time
import requests

def call_api_with_exponential_backoff(url: str, max_retries: int = 5, initial_delay: float = 1.0) -> dict:
    """
    Executes an HTTP GET request with exponential backoff logic.
    Doubles the delay after each failed attempt to prevent service overloading.
    """
    delay = initial_delay
    
    for attempt in range(1, max_retries + 1):
        try:
            response = requests.get(url, timeout=5)
            response.raise_for_status()  # Raises HTTPError for 4xx/5xx codes
            return response.json()
        except (requests.RequestException, Exception) as err:
            print(f"[Attempt {attempt}/{max_retries}] Failed: {err}. Retrying in {delay} seconds...")
            if attempt == max_retries:
                raise RuntimeError(f"API call failed after {max_retries} attempts.") from err
            
            time.sleep(delay)
            delay *= 2  # Exponential escalation (1s, 2s, 4s, 8s...)

# Architectural Explanation provided to Interviewer:
# Exponential backoff prevents "thundering herd" problems during transient API outages or rate limits.
# In production, adding random jitter (e.g., delay + random.uniform(0, 0.5)) prevents synchronized retry spikes.

```

---

### Task 2: Implement an In-Memory Cache for LLM Queries to eliminate duplicate API calls for identical inputs.

```python
import hashlib
from typing import Optional

class LLMQueryCache:
    """
    In-memory caching layer for LLM responses using SHA-256 hashed queries as keys.
    """
    def __init__(self):
        self._cache: dict[str, str] = {}

    def _generate_key(self, prompt: str) -> str:
        # Standardize formatting and hash string to fixed key length
        normalized_prompt = prompt.strip().lower()
        return hashlib.sha256(normalized_prompt.encode('utf-8')).hexdigest()

    def get(self, prompt: str) -> Optional[str]:
        key = self._generate_key(prompt)
        return self._cache.get(key)

    def set(self, prompt: str, response: str) -> None:
        key = self._generate_key(prompt)
        self._cache[key] = response

# Example Usage
cache = LLMQueryCache()
user_prompt = "What is the capital of France?"

# First execution (Cache Miss)
cached_res = cache.get(user_prompt)
if cached_res is None:
    # Simulate LLM Call
    llm_output = "The capital of France is Paris."
    cache.set(user_prompt, llm_output)
    print("Fetched from LLM API.")
else:
    print(f"Cache Hit: {cached_res}")

# Second execution (Cache Hit)
cached_res = cache.get(user_prompt)
if cached_res:
    print(f"Cache Hit: {cached_res}")

# Architectural Explanation provided to Interviewer:
# Direct string hashing provides exact-match caching with O(1) lookup time.
# Production Enhancement: Replace exact matching with Semantic Caching (e.g., Redis VL / GPTCache) 
# using embedding similarity thresholds (e.g., cosine distance < 0.05) to catch rephrased queries.

```

```

```
