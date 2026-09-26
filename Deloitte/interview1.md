# Deloitte Generative AI Engineer Interview (2026-2027) — Technical Round Questions & Answers

---

## 1. Project Discussion & Fundamental RAG Mechanics

### Question 1: Tell me about a Generative AI project that you have worked on recently.

**Answer:**

* **Core Application:** Built an enterprise document chatbot utilizing a Retrieval-Augmented Generation (RAG) architecture.
* **Pipeline Components:** Document ingestion, chunking, embedding generation using standard transformer models, vector database indexing, dynamic similarity search, and prompt synthesis powered by an LLM backend.

---

### Question 2: If ChatGPT / frontier LLMs are so powerful and have huge context windows, why don't we just send the entire PDF directly to the model instead of using RAG?

**Answer:**

1. **Context Window Limitations & Cost:** Sending full multi-page PDFs repeatedly quickly exceeds or bloats token limits, driving up API costs exponentially with every user query.
2. **Retrieval Efficiency & Latency:** Large context payloads significantly increase Time-To-First-Token (TTFT) and processing latency.
3. **Information Needle-in-a-Haystack Problem:** LLMs tend to perform worse at retrieving precise facts buried deep in massive prompt contexts ("lost in the middle" phenomenon) compared to focused, relevant context chunks provided by RAG.

---

### Question 3: What chunk size do you usually use, and why add overlap between chunks?

**Answer:**

* **Chunk Size:** Typically between **500 to 1,000 tokens**, dynamically adjusted depending on document density and structural complexity.
* **Why Add Overlap (e.g., 10–20% / 50–100 tokens):**
* Prevents critical semantic context from being sliced across boundary splits.
* Preserves continuity of ideas, entities, and relationships between adjacent text segments so key details are not lost during vector retrieval.



---

### Question 4: Explain embeddings without using the word "vector."

**Answer:**

* An embedding converts human text into a **mathematical or numerical representation** so that words, sentences, or concepts with similar meanings can be identified, grouped, and compared efficiently based on their semantic relationships.

---

## 2. Debugging, Latency & Production Scenarios

### Question 5: Imagine your chatbot is giving wrong answers to users. What would be your debugging approach?

**Answer:**
Instead of immediately changing or blaming the underlying LLM, systematically isolate components across the pipeline:

1. **Verify Document Ranking & Retrieval Quality:** Inspect if the vector store actually fetched the correct candidate chunks for the query.
2. **Assess Source Relevance:** Check whether the source documents in the knowledge base contain accurate, up-to-date information.
3. **Evaluate Prompt Construction:** Ensure system instructions and retrieved context are clearly structured without conflicting guidelines.
4. **Inspect LLM Generation Parameters:** Check settings like temperature or top-p if output formatting or reasoning is drifting.

---

### Question 6: Can hallucinations be completely removed from a Generative AI application?

**Answer:**

* **No.** Hallucinations cannot be 100% eliminated because LLMs are fundamentally probabilistic next-token prediction engines rather than deterministic fact lookup tables.
* **Mitigation Strategy:** Hallucinations can be significantly reduced using strict RAG grounding, lower temperature settings, refusal constraints (*"I don't know"*), prompt guardrails, and automated groundedness validation steps before outputs reach the end user.

---

### Question 7: Walk through everything that happens behind the scenes when a user submits a query in your RAG application, and identify where maximum latency occurs.

**Answer:**

* **Behind-the-Scenes Pipeline Flow:**
1. User submits natural language query.
2. Query is converted into an embedding representation via an embedding API/model.
3. Similarity search runs against the vector database (e.g., ANN search via HNSW index).
4. Top-$K$ relevant context chunks are retrieved.
5. Context chunks + system prompt + user query are assembled.
6. Assembled prompt payload is sent to the LLM.
7. LLM processes context and generates the completion response.


* **Maximum Latency Bottleneck:** The **LLM generation stage** (specifically generation time and Time-To-First-Token) introduces the vast majority of latency in the pipeline, followed by embedding generation and network I/O. Vector database retrieval is typically the fastest phase when indexed properly.

---

## 3. LLM Parameters & Enterprise Security

### Question 8: What happens when temperature is set to 0, and what if it is increased to 1.2?

**Answer:**

* **Temperature = 0:** The model becomes deterministic and focused, picking the highest-probability token at each step. This yields consistent, highly repeatable outputs ideal for factual extraction and structured tasks.
* **Temperature = 1.2:** Introduces high randomness into the token probability distribution. Creativity and variety increase significantly, but reliability, factual adherence, and consistency decrease, raising the risk of nonsensical outputs or hallucinations.

---

### Question 9: Scenario: A client states that their company data must never leave their enterprise environment. What Generative AI solution architecture would you propose?

**Answer:**

* **Enterprise Cloud Private Deployments:** Deploy dedicated, isolated model instances (e.g., Azure OpenAI Service with private endpoints) where data is not logged, retained, or used for model training.
* **Self-Hosted Open-Source Models:** Host open-weights models (e.g., Llama 3, Mistral) on self-managed infrastructure within the client's private cloud (VPC) or on-premise servers.
* **Enterprise Security Measures:**
* Strict Role-Based Access Control (RBAC) and identity provider integration (e.g., Azure AD / Okta).
* Data encryption at rest and in transit.
* Private VPC peering to ensure zero data egress to public internet endpoints.



---

## 4. Enterprise Architecture & Scalability System Design

### Question 10: System Design: Design a corporate document chatbot that serves 10,000 employees and answers questions from internal company files. How would you scale this system if usage suddenly increases 10x?

**Answer:**

```
                  [ 10,000+ Corporate Employees ]
                                 │
                                 ▼
                     [ API Gateway & Auth ]
                                 │
                     [ Async Load Balancer ]
                                 │
                 ┌───────────────┴───────────────┐
                 ▼                               ▼
       [ Semantic Redis Cache ]        [ FastAPI Services ]
          (Cached Answers)                       │
                                                 ├──> [ Async Vector DB Cluster ]
                                                 │    (Sharded Index + Metadata Filters)
                                                 │
                                                 └──> [ Managed / Self-Hosted LLM Pool ]
                                                      (Token Rate Limit Queues)

```

1. **Core Architecture Layers:**
* **Ingestion Layer:** Document parsing, semantic chunking, embedding worker pools, delta indexing.
* **Retrieval Layer:** Distributed vector database with metadata filtering scoped to employee authorization groups.
* **LLM & API Layer:** Async API services (FastAPI) backed by load-balanced LLM endpoints.
* **Security & Observability Layer:** Single Sign-On (SSO/OAuth2), RBAC access control, trace monitoring, audit logging.


2. **Scaling 10x Strategy (System Design):**
* **Semantic Caching:** Deploy a distributed cache (e.g., Redis) using vector similarity to serve repeated corporate queries instantly, bypassing vector search and LLM invocation.
* **Vector Database Sharding & Indexing:** Distribute vector indices across multi-node clusters, utilize HNSW indexing, and pre-filter searches using metadata tags to keep query latencies low.
* **Async Request Queues:** Implement worker queues (e.g., Celery/Redis) with auto-scaling API containers to absorb query spikes.
* **Model Provisioning & Fallbacks:** Provision dedicated throughput/capacity (e.g., Provisioned Throughput Units on Azure OpenAI) with fallback routes to secondary model regions to prevent hitting rate limits during peak usage hours.
