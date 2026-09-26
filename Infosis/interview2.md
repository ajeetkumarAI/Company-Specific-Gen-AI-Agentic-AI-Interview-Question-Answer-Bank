# Infosys Generative AI Engineer Interview — Questions & Answers Guide

---

## 1. Project Deep Dive & Production Considerations

### Question 1: What were the key aspects of your project deep dive (problem statement, tech stack selection, deployment, target audience size, and document types)?

**Answer:**

* **Problem Statement & Scope:** Solved enterprise knowledge retrieval challenges by building a enterprise hybrid system blending standard RAG (50%) with Agentic AI capabilities (50%) for autonomous task execution.
* **Tech Stack Selection:**
* **Frameworks:** LangChain / LlamaIndex for RAG pipelines; LangGraph / CrewAI for multi-agent workflows.
* **Vector DB:** Distributed cloud-managed vector stores (e.g., Pinecone / PGVector) with metadata filtering.
* **LLM Engine:** Azure OpenAI / vLLM-hosted models integrated via asynchronous endpoints.


* **Production Deployment & Audience Scale:** Hosted via containerized microservices (FastAPI on Kubernetes) with autoscaling policies to serve thousands of concurrent active employees with low tail latency.
* **Document Types & Processing:** Handles heterogeneous enterprise inputs including structured tables, multi-page PDFs, scanned images, and raw text formats.

---

### Question 2: How do document types and data volume impact cloud deployment costs and infrastructure architecture in RAG applications?

**Answer:**

* **Parsing & Extraction Costs:** Complex document types (scanned PDFs, multi-column tables) require OCR and heavy parsing engines (e.g., Unstructured, LlamaParse) which increase compute overhead during ingestion compared to plain text.
* **Storage & Vector Indexing:** High document volumes increase vector storage footprints linearly. Indexing algorithms like HNSW require significant memory (RAM) for fast retrieval, driving up cloud infrastructure instance costs.
* **Deployment Optimization Strategy:**
* Implement **incremental/delta ingestion pipelines** so only modified or newly uploaded files are chunked and embedded.
* Apply **pre-retrieval metadata pre-filtering** to shrink search scopes and lower CPU/GPU compute during vector queries.



---

## 2. Technical RAG Mechanics & Prompt Construction

### Question 3: Walk through an end-to-end RAG pipeline, including document loading frameworks and chunking strategies.

**Answer:**

1. **Document Ingestion & Loading:** Extract text using specialized document loaders (e.g., `PyPDFLoader`, `UnstructuredPDFLoader`) to preserve document structure.
2. **Chunking Strategy:**

* **Recursive Character Chunking:** Splits text hierarchically using double newlines, single newlines, and spaces to preserve natural structural units (e.g., 500–1000 tokens with 10% overlap).
* **Semantic Chunking:** Computes embedding distance between consecutive sentences and creates boundary splits when semantic shift exceeds a set threshold.

3. **Embedding & Indexing:** Dense vectors generated via embedding models are indexed into a vector database using Approximate Nearest Neighbor (ANN) indexing (e.g., HNSW).
4. **Retrieval & Reranking:** Query is embedded, top-$K$ chunks are retrieved via vector search, and a cross-encoder reranker sorts candidate chunks by relevancy.
5. **Generation:** Re-ranked chunks are passed alongside the user query to the LLM to generate a grounded output.

---

### Question 4: What is the difference between a System Prompt and a User Prompt?

**Answer:**

* **System Prompt:** Sets the core behavior, role identity, capabilities, tone, constraints, safety guardrails, and output rules for the LLM (e.g., *"You are an HR policy assistant. Answer ONLY using the provided documents."*). It remains constant across user sessions.
* **User Prompt:** The dynamic, variable natural language input or query provided by the end user during an active session (e.g., *"What is the annual leave rollover policy?"*).

---

## 3. Python Core Concepts & Multithreading

### Question 5: What is the difference between Multithreading and Multiprocessing in Python? How does GIL affect them?

**Answer:**

* **Multithreading (`threading`):** Runs multiple threads within a single process sharing the same memory space.
* **GIL Impact:** Python's **Global Interpreter Lock (GIL)** prevents multiple native threads from executing Python bytecode simultaneously. Therefore, multithreading provides speedups primarily for **I/O-bound tasks** (e.g., network API requests, database queries, file reading).


* **Multiprocessing (`multiprocessing`):** Spawns distinct Python processes, each with its own Python interpreter and memory space.
* **GIL Impact:** Bypasses the GIL entirely, allowing true parallel execution across multi-core CPUs. Ideal for **CPU-bound tasks** (e.g., heavy data transformations, tokenization, text parsing, local matrix computations).



---

### Question 6: What is GIL (Global Interpreter Lock) in Python, and why is it important for Gen AI backend systems?

**Answer:**

* **Definition:** GIL is a mutual-exclusion lock used by the CPython interpreter to synchronize thread execution and ensure thread safety when managing object memory.
* **Relevance to Gen AI Backends:** Since Gen AI backend services handle heavy I/O operations (fetching embeddings, hitting external LLM APIs, querying vector databases), multithreading or asynchronous frameworks (`asyncio`) remain highly efficient despite the GIL. However, CPU-heavy tasks like document parsing or embedding calculation should be offloaded to process pools (`ProcessPoolExecutor`) or worker queues (Celery) to prevent GIL bottlenecks.

---

## 4. Hallucinations, Evaluation & Monitoring

### Question 7: Why do LLMs hallucinate? What parameters and factors contribute to hallucinations?

**Answer:**

* **Why LLMs Hallucinate:** LLMs generate responses by computing token probability distributions, not by querying an internal truth engine.
* **Contributing Factors & Parameters:**
* **High Temperature / High Top-P:** Flattens probability distributions, forcing the model to sample lower-probability tokens.
* **Poor Context Quality / Noise:** Irrelevant, truncated, or conflicting retrieved chunks in a RAG prompt lead to fabricated synthesis.
* **Context Length Limits:** Extremely long prompts can result in the model ignoring critical details buried in the middle of context ("lost in the middle").



---

### Question 8: How do you evaluate and monitor the performance of an LLM / RAG pipeline in production?

**Answer:**

* **Evaluation Metrics (RAGAS / DeepEval):**
* **Faithfulness:** Measures whether the generated answer relies *strictly* on retrieved context without outside inventions.
* **Answer Relevance:** Evaluates how directly the output addresses the user query.
* **Context Precision & Recall:** Measures how relevant the retrieved context chunks are relative to the true ground truth.


* **Production Monitoring Tools & Frameworks:**
* Deploy observability platforms like **LangSmith**, **Phoenix (Arize)**, or **OpenTelemetry** to trace every step of the pipeline.
* Monitor real-time telemetry metrics: Time-To-First-Token (TTFT), total token consumption, latency per component, error/refusal rates, and user feedback signals (thumbs up/down).
