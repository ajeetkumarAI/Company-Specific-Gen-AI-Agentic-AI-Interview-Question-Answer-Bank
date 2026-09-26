# EY Generative AI Engineer Interview (2026-2027) — Questions & Answers Guide

---

## 1. Technical Scenario Round: Production Debugging & System Design

### Question 1: Scenario: Your RAG-based chatbot for a client is confidently returning information that does not exist in the knowledge base (hallucination). The client is upset. How do you diagnose and fix this in production today?

**Answer:**

1. **Isolate Retrieval vs. Generation Failure:**
* Determine if the retrieval step failed to fetch the right context chunks, or if the LLM received the correct context but ignored/hallucinated beyond it.
* Inspect retrieval similarity scores and log the raw context chunks passed to the LLM for failing queries.


2. **Mitigate LLM Hallucinations (Prompt Engineering & Grounding Constraints):**
* Tighten system prompts with explicit **grounding instructions** and strict refusal rules (e.g., *"Answer ONLY using the provided context. If the answer is not present, state 'I do not have this information'"*).


3. **Implement Light Guardrails & Automated Checks:**
* Introduce a fast, cheap **groundedness check** (a lightweight classifier or small model evaluator) that compares the generated response against retrieved chunks before serving it to the user.


4. **Immediate Fallback Mechanism:**
* If the groundedness score falls below a set safety threshold, intercept the output and trigger a safe fallback message (*"I cannot find this information in the official documents"*) rather than serving confident, false statements.



---

### Question 2: Scenario: Your knowledge base expanded from 10,000 documents to 5 million documents. Latency has crashed, and API/infrastructure costs are spiking. How do you scale this architecture?

**Answer:**

1. **Vector Database Optimization & Indexing:**
* Migrate to a production-grade, distributed vector database (e.g., Pinecone, Weaviate, or tuned PGVector).
* Tune **Approximate Nearest Neighbor (ANN)** index parameters (HNSW/IVF) to trade a negligible fraction of recall for massive speed gains.


2. **Pre-Filtering with Metadata:**
* Implement **metadata pre-filtering** so the vector search space is partitioned before dense similarity execution rather than filtered post-retrieval.


3. **Tiered / Hybrid Retrieval Architecture:**
* Implement a two-tiered retrieval pipeline: run a fast, low-cost sparse keyword search (BM25) first to narrow millions of docs down to candidate subsets, then apply dense vector search and cross-encoder reranking on top candidates.


4. **Caching Strategy:**
* Implement semantic caching (e.g., Redis) to serve repeated or semantically equivalent user queries directly from cache without hitting the vector database or LLM.



---

### Question 3: Scenario: Your Gen AI feature was approved for a $2,000/month API budget, but costs hit $18,000/month and are rising. How do you analyze and optimize cost consumption?

**Answer:**

1. **Model Routing & Cascading:**
* Replace one-size-fits-all calls to expensive frontier models (e.g., GPT-4 class) with **intelligent routing**. Route simple extraction, classification, or summarization tasks to smaller, lower-cost models (e.g., GPT-4o-mini, Claude Haiku, or fine-tuned SLMs).


2. **Prompt Payload Reduction:**
* Inspect prompt sizes to ensure full, un-chunked raw documents are not being injected into context when small, precise retrieved chunks suffice.


3. **Caching & Deduplication:**
* Enable **prompt caching** provided by LLM vendors and deploy semantic caching at the backend layer to eliminate duplicate LLM calls.


4. **Batch Processing:**
* Offload non-real-time asynchronous tasks (e.g., nightly report generation, data indexing summary jobs) to batch processing APIs, which typically offer 50% cost discounts.



---

## 2. Case Study Round: Enterprise RAG Architecture Design

### Question 4: Case Study: Design an internal AI tool for employees to query 3,000 HR and policy PDF documents (updated weekly). Requirements: Strict multi-department access control (e.g., Legal can only view Legal policies, HR can only view HR policies; zero data leakage between departments). Walk through the end-to-end design.

**Answer:**

```
[ Ingestion Pipeline ]
  PDF Documents (3,000) ──> Chunking ──> Tag Metadata (dept_id) ──> Delta Re-Embedding (Weekly Changes)

[ Query & Retrieval Pipeline ]
  User Query + User Identity (RBAC) ──> Pre-Retrieval Metadata Filter ──> Scoped Vector Search ──> Relevant Chunks

[ Generation & Guardrail Layer ]
  Scoped Chunks ──> System Prompt + Citation Instructions ──> Grounded Response with Citations

[ Continuous Evaluation ]
  Periodic Automated Test Suite ──> Check Answer Accuracy & Verify Access Control / Zero Leakage

```

1. **Ingestion Layer:**
* Chunk PDF documents and attach strict **department access metadata tags** (`dept_id`, `access_level`) to every vector chunk at ingestion time.
* Implement **delta/incremental re-indexing** for weekly document updates, re-embedding only added or modified PDFs instead of re-processing the entire 3,000-document store.


2. **Pre-Retrieval Access Control (Critical):**
* Enforce **pre-retrieval filtering**: the vector database query must be scoped directly to match the requesting user's validated RBAC (Role-Based Access Control) permissions.
* *Never rely on the LLM prompt to restrict or hide sensitive data.* Unfiltered context chunks must never enter the model prompt if the user lacks explicit permissions.


3. **Generation Layer:**
* Inject retrieved, permission-verified chunks into the prompt with strict formatting rules requiring explicit source document citations so users can verify outputs.


4. **Evaluation & Compliance Monitoring:**
* Run continuous test suites per department to systematically verify output accuracy and test against permission bypass/data leakage risks, treating access bugs as critical compliance violations.



---

### Question 5: Follow-Up: How do you handle overlapping policies (e.g., a document relevant to both Legal and HR departments) without duplicating data?

**Answer:**

* **Multi-Tag Metadata Association:** Tag overlapping document chunks with a multi-value array or list of authorized departments (`departments: ["Legal", "HR"]`) instead of duplicating storage entries.
* **Filter Operator Logic:** Update the pre-retrieval vector search filter condition from an exact match (`department == user_dept`) to an inclusion match (`user_dept IN metadata.departments`).

---

## 3. Client-Facing & Behavioral Round

### Question 6: Client Scenario: A non-technical client insists their chatbot must be 100% accurate and "never make a mistake." How do you explain that this expectation is unrealistic without losing their confidence?

**Answer:**

* **Avoid Technical Jargon:** Frame the explanation around **risk management and operational guardrails** rather than raw AI limitations or model mathematics.
* **Analogy & Reality Framing:** Explain that no complex system—human or AI—is 100% error-free. Just as human specialists require review processes, AI systems require governance structures.
* **Focus on Containment & Guardrails:** Shift the conversation to how the engineering team mitigates risks:
* Automated groundedness filters.
* Explicit fallback routines when confidence is low.
* Human-in-the-Loop escalation paths for high-stakes decisions.



---

### Question 7: Standard Behavioral Scenarios

* **Scope Changes Mid-Project:** Handled by prioritizing core deliverables, evaluating trade-offs with stakeholders, establishing clear change-control documentation, and re-baselining delivery timelines transparently.
* **Technical Disagreements with Teammates:** Resolved through objective benchmarking, running isolated proof-of-concept (PoC) tests, comparing metric trade-offs (e.g., latency vs. accuracy vs. cost), and aligning on common enterprise goals.
* **Staying Up-to-Date with AI Advances:** Continuously follow research papers, major open-source repository releases, hands-on prototyping, and industry-standard engineering benchmarks.
