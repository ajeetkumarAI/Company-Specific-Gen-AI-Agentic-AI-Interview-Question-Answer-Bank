# EY (Ernst & Young) Generative AI Engineer Interview Guide

---

## 1. Candidate Background & Loop Summary

* **Role & Company:** Generative AI Engineer at EY (Client-Facing Consulting Role).
* **Interview Loop Structure (4 Rounds):**
  1. **Recruiter Screening:** Initial background check, compensation alignment, and logistics (Standard).
  2. **Technical Scenario & Troubleshooting Round (45 Mins):** Whiteboard/oral scenario reasoning focused on live incident response, production debugging, latency/cost optimization, and scaling.
  3. **Enterprise Case Study Round (60 Mins):** End-to-end system design of an internal HR policy chatbot serving 3,000+ PDFs across departments with zero security/access-control leaks.
  4. **Client-Facing & Behavioral Round:** Translating complex technical constraints (e.g., non-determinism, hallucinations) to non-technical stakeholders, handling scope changes, and managing technical disagreements.

---

## 2. Technical Scenarios & Live Production Incident Response

### Scenario 1: A production RAG chatbot is confidently hallucinating information that does not exist in the knowledge base. The client is escalatng. How do you diagnose and fix this in production today?

#### Diagnostic & Fix Workflow


```

```
               [ Production Incident: High Hallucination ]
                                   │
                                   ▼
                     [ Isolate Failure Point ]
                                   │
        ┌──────────────────────────┴──────────────────────────┐
        ▼                                                     ▼

```

[ Retrieval Failure Check ]                           [ Generation Failure Check ]
(Query vector logs: Are valid                        (Are valid chunks present in prompt,
chunks retrieved in top-K?)                          but ignored by the model?)
│                                                     │
▼                                                     ▼
• Apply Hybrid Search (BM25 + Dense)                  • Enforce Strict System Guardrails
• Adjust chunking parameters                          • Add Groundedness Evaluator
• Tune HNSW index recall                              • Trigger Fallback ("I don't know")

```

1. **Isolate Retrieval Failure vs. Generation Failure:**
   * Query the production vector logs for the failing requests.
   * If the correct context chunks are **not** present in the top-$K$ candidates, the issue is a **retrieval failure** (fix chunking, embeddings, or search scoring).
   * If correct chunks **are** present but the model invents external facts, the issue is a **generation failure** (model ungroundedness).

2. **Immediate Production Mitigation (Generation Failure):**
   * **Prompts & Boundaries:** Tighten the system prompt with explicit constraints (e.g., *"Answer ONLY using the provided `<context>` tags. If the exact answer is not contained within, respond with 'I do not have sufficient information to answer this question.'"*).
   * **Groundedness Guardrail (LLM-as-a-Judge / Light Classifier):** Insert an inline check (e.g., automated entailment step or small classifier) between model output generation and user response delivery.

3. **Fallback Implementation:**
   * If the calculated groundedness score falls below a safety threshold (e.g., $<0.85$), suppress the completion and return a safe fallback: *"I am sorry, but I cannot find a verified reference in the knowledge base to answer this query."*

> **Key Takeaway:** It is significantly better for a client-facing consulting solution to under-perform visibly with a safe fallback than to deliver false information confidently.

---

### Scenario 2: The knowledge base grew from 10,000 documents to 5,000,000 documents. Retrieval latency crashed, and hosting costs spiked. How do you re-architect the system?

#### Architectural Interventions
1. **Metadata Pre-Filtering:** Do not run raw vector similarity across the entire 5-million vector index. Apply structured metadata pre-filters (e.g., `tenant_id`, `department`, `document_type`, `date_range`) to shrink the search space before vector computation occurs.
2. **HNSW Index Parameter Tuning:**
   * Adjust **HNSW (Hierarchical Navigable Small World)** parameters such as $M$ (number of bi-directional links per node) and `efSearch` (size of the dynamic candidate list during search) to trade off fractional recall gains for massive speed improvements.
3. **Tiered / Two-Stage Retrieval:**
   * **Stage 1 (Coarse & Cheap):** Execute BM25 / Sparse keyword search or inverted index lookup to trim the 5,000,000 document space down to top 500 candidate chunks.
   * **Stage 2 (Fine & Semantic):** Run dense vector similarity and cross-encoder reranking *only* over those top 500 candidate chunks.
4. **Vector Database Sharding & Semantic Caching:** Migrate to scalable vector databases (e.g., Pinecone, Weaviate, Qdrant, or sharded pgvector) and implement a **Semantic Cache (Redis)** to serve frequent user queries instantly without hitting vector indices or LLM APIs.

---

### Scenario 3: A Gen AI feature was approved for a $2,000/month LLM API budget. Current spend is hitting $18,000/month and rising. How do you audit and optimize this?

#### Cost Audit & Optimization Strategy
1. **Model Tiering & Dynamic Query Routing:** Stop using frontier models (e.g., GPT-4o, Claude 3.5 Sonnet) for basic classification, summarization, or simple retrieval synthesis. Route requests dynamically based on query complexity:
   * **Simple Queries:** Route to smaller, distilled models (e.g., GPT-4o-mini, Llama 3.1 8B, Claude 3 Haiku).
   * **Complex Queries:** Route to high-reasoning frontier models.
2. **Context Window Optimization:** Audit prompt construction logs. Ensure entire multi-page documents are not passed in every API payload when only 3 retrieved chunks (e.g., 1,200 tokens) are necessary.
3. **Eliminate Duplicate API Calls via Caching:** Implement exact-match prompt caching and semantic response caching (e.g., Redis VL, GPTCache) to intercept identical or highly similar user queries.
4. **Batch Processing:** For non-real-time workloads (such as nightly evaluation runs or document summarizations), utilize OpenAI / Anthropic **Batch APIs** to obtain a 50% cost discount.

---

## 3. Case Study: Enterprise HR Chatbot System Design

### Client Requirements
* **Scale:** 3,000 internal HR PDF documents, updated weekly.
* **Security & Isolation:** Role-Based Access Control (RBAC)—Legal team must only access Legal documents; HR team must only access HR documents. Zero cross-department information leakage allowed.

---

### Complete System Design


```

```
         ┌─────────────────────────────────────────────────────────────┐
         │                     INGESTION PIPELINE                      │
         └─────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
                       [ Weekly Modified Document Loader ]
                                        │
                                        ▼
                       [ Chunking & Department Metadata Tagging ]
                         (e.g., metadata={"dept": ["HR", "Legal"]})
                                        │
                                        ▼
                       [ Embedded & Indexed Vector Store ]
                                        │
                                        │
         ┌──────────────────────────────┴──────────────────────────────┐
         │                     RETRIEVAL & SECURITY                    │
         └─────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
                       [ User Authenticated JWT Input ]
                                        │
                                        ▼
                       [ Pre-Retrieval Metadata Filter ]
                         (Query scope restricted BEFORE vector search)
                                        │
                                        ▼
                       [ Scoped Vector Database Search ]
                                        │
                                        ▼
                       [ Cross-Encoder Reranker ]
                                        │
                                        │
         ┌──────────────────────────────┴──────────────────────────────┐
         │                    SYNTHESIS & EVALUATION                   │
         └─────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
                       [ Grounded LLM Prompt Generation ]
                         (With document citations)
                                        │
                                        ▼
                       [ Offline Evaluation Suite (RAGAS) ]

```

```

---

### Layer-by-Layer Breakdown

#### 1. Ingestion Layer
* **Delta Processing:** Instead of re-indexing 3,000 documents weekly, run a file hash comparison (SHA-256) during the weekly cron job to process and embed **only newly added or updated files**.
* **Metadata Attachment:** Attach explicit access arrays during chunking:
  ```json
  {
    "chunk_id": "pdf_882_chunk_4",
    "text": "Medical leave rollover policies...",
    "allowed_departments": ["HR", "Legal"]
  }

```

#### 2. Retrieval & Security Layer (Pre-Retrieval RBAC)

* **Pre-Search Filtering:** Never rely on system prompts to enforce data access boundaries (e.g., *"Do not reveal legal docs to HR"*).
* **Scoped Execution:** Embed user identity assertions (JWT tokens) at the API Gateway level. Filter vector searches directly at the database query layer:
```python
# Scoped Vector Search Example
user_department = user_jwt.get("department")  # e.g., "HR"

vector_db.query(
    vector=query_embedding,
    top_k=5,
    filter={"allowed_departments": {"$in": [user_department]}}
)

```


* **Overlapping Policies Handling:** If a document applies to both HR and Legal, tag `allowed_departments` with `["HR", "Legal"]` and use `$in` operator matching instead of exact single-string equality.

#### 3. Generation & Citation Layer

* Enforce strict inline source citation mapping (`[Doc Name, Page Number]`) inside the prompt template so employees can independently verify retrieved policies against raw PDFs.

#### 4. Evaluation & Compliance Layer

* Construct a benchmark evaluation set across departments. Run automated nightly regression pipelines testing for both **Answer Correctness** (RAGAS) and **Security Leakage** (attempting cross-department unauthorized queries).

---

## 4. Client-Facing Communication & Behavioral Scenarios

### Scenario 1: The client insists their chatbot must be 100% accurate and never make a mistake. How do you handle this without jargon?

#### Approach & Script

Avoid deep neural network theory, mathematical loss functions, or probability distribution jargon. Frame the conversation entirely around **Risk Management and Containment**:

> *"In enterprise software, no automated system—or human workforce—operates at 100% flawless accuracy. Generative models operate probabilistically, similar to hiring a highly capable analyst.*
> *Our engineering focus is not promising theoretical perfection; it is building **Guardrails, Isolation Filters, and Monitoring Loops** around the system. When the model encounters an ambiguous request or an out-of-scope query, our guardrails catch it, prevent raw guessing, and fail safely by routing the user to a human operator or returning a verified fallback response."*

---

### Core Interview Themes for EY Consulting Roles

1. **Deterministic Security Over Model Promises:** Never trust an LLM to enforce access control or privacy rules through system prompt instructions alone. Enforce security hard-stops at the infrastructure/database metadata layer.
2. **Translate Limitations to Risk Trade-offs:** When communicating technical limitations to clients, frame them in terms of business impact, risk containment, and operational fallback mechanisms.
3. **Close Every System Design with Telemetry:** Conclude system design answers by stating how you will measure, evaluate (e.g., RAGAS/DeepEval), and trace (e.g., LangSmith/Phoenix) the proposed fix in production.

---

## 5. Live Python Challenge: Enterprise RAG Cost Router & Guardrail Engine

```python
import hashlib
import time
from typing import Dict, Any, Optional

class EnterpriseRAGRouter:
    """
    Production-grade Query Router, Caching Engine, and Cost Guardrail.
    Demonstrates dynamic model selection, query sanitization, and fallback rules.
    """
    def __init__(self, cost_threshold_usd: float = 0.05):
        self.cost_threshold = cost_threshold_usd
        self.response_cache: Dict[str, str] = {}
        
    def _hash_prompt(self, prompt: str) -> str:
        """Generates SHA-256 key for exact prompt caching."""
        return hashlib.sha256(prompt.strip().lower().encode('utf-8')).hexdigest()

    def route_query_by_complexity(self, user_query: str) -> Dict[str, Any]:
        """
        Dynamically selects model tier based on token length and query scope
        to optimize cloud LLM spend.
        """
        word_count = len(user_query.split())
        
        # Simple/Short query -> Light, inexpensive model
        if word_count < 15 and not any(kw in user_query.lower() for kw in ["analyze", "compare", "synthesize"]):
            return {
                "selected_model": "gpt-4o-mini",
                "estimated_cost_per_1k_tokens": 0.00015,
                "tier": "Low-Cost Tier"
            }
        # Complex query -> High-capacity reasoning engine
        else:
            return {
                "selected_model": "gpt-4o",
                "estimated_cost_per_1k_tokens": 0.0025,
                "tier": "Frontier Tier"
            }

    def execute_rag_pipeline(self, user_query: str, user_dept: str, retrieved_chunks: list) -> str:
        """
        Executes query caching, groundedness check, and fallback isolation.
        """
        cache_key = self._hash_prompt(f"{user_dept}:{user_query}")
        
        # 1. Exact Cache Hit
        if cache_key in self.response_cache:
            return f"[CACHE HIT] {self.response_cache[cache_key]}"
        
        # 2. Check Retrieval Groundedness
        if not retrieved_chunks:
            # Fallback instead of allowing hallucination
            return "I am sorry, but I do not have access to verified context to answer this query."
        
        # 3. Determine Routing
        routing_info = self.route_query_by_complexity(user_query)
        
        # Simulate LLM Generation
        generated_response = (
            f"[Served via {routing_info['selected_model']}] "
            f"Based on policy docs: {retrieved_chunks[0]}"
        )
        
        # Update Cache
        self.response_cache[cache_key] = generated_response
        return generated_response

# --- Example Production Execution ---
if __name__ == "__main__":
    router = EnterpriseRAGRouter()
    
    # Example 1: Missing context triggers safe fallback
    res1 = router.execute_rag_pipeline(
        user_query="What is the 2026 sabbatical policy?",
        user_dept="HR",
        retrieved_chunks=[]
    )
    print("Execution 1:", res1)
    
    # Example 2: Valid context triggers dynamic low-cost routing
    res2 = router.execute_rag_pipeline(
        user_query="What is the travel allowance limit?",
        user_dept="Finance",
        retrieved_chunks=["Travel allowance limit is $100 per day for meals."]
    )
    print("Execution 2:", res2)
    
    # Example 3: Subsequent identical request hits cache
    res3 = router.execute_rag_pipeline(
        user_query="What is the travel allowance limit?",
        user_dept="Finance",
        retrieved_chunks=["Travel allowance limit is $100 per day for meals."]
    )
    print("Execution 3:", res3)

```
