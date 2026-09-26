# TCS Generative AI Engineer Interview — Questions & Answers Guide

---

## 1. LLM Fundamentals, Hyperparameters & Model Architectures

### Question 1: What is the difference between Top-K and Top-P sampling? Where are they used, and what are their real-life use cases?

**Answer:**

* **Top-K Sampling:** Filters the candidate token pool to strictly the top $K$ most probable next tokens, redistributing probability among only those $K$ options.
* *Use Case:* Useful when you want tight boundaries over generation choices, but can produce repetitive output if $K$ is too low or include low-quality tokens if $K$ is static across varying probability distributions.


* **Top-P (Nucleus) Sampling:** Dynamically selects the smallest set of top tokens whose cumulative probability mass equals or exceeds threshold $P$ (e.g., $0.90$).
* *Use Case:* Highly adaptive across varying contexts. In predictable contexts, the candidate pool shrinks automatically; in creative contexts, it expands naturally.


* **Real-World Application:** Factual corporate extraction or code generation uses low $P$ ($0.1–0.3$) or small $K$, whereas creative copywriting or brainstorming tasks use higher $P$ ($0.8–0.95$).

---

### Question 2: What is the Transformer architecture, and why do LLMs rely on it instead of traditional RNNs?

**Answer:**

* **Transformer Architecture:** Introduced in "Attention Is All You Need," transformers rely entirely on the **Self-Attention Mechanism** to model dependencies between tokens across an entire sequence simultaneously without sequential recurrence.
* **Why Transformers Replace RNNs:**
* **Parallelization:** RNNs process text sequentially (token-by-token), creating a severe hardware bottleneck during training. Transformers process full context sequences in parallel on modern GPUs.
* **Long-Range Dependencies:** RNNs suffer from vanishing/exploding gradients over long contexts. Transformers compute direct pairwise attention weights between all tokens regardless of sequence distance.



---

### Question 3: What is the difference between Decoder-Only models and Encoder-Decoder models?

**Answer:**

* **Decoder-Only Models (e.g., GPT-4, Llama):**
* Uses causal self-attention (tokens can only attend to previous tokens).
* *Best For:* Autoregressive text generation, open-ended conversation, reasoning, and code completion.


* **Encoder-Decoder Models (e.g., T5, BART):**
* The encoder processes the full source sequence bidirectionally; the decoder generates the output autoregressively while cross-attending to the encoded representation.
* *Best For:* Sequence-to-sequence transformation tasks like language translation, summarization, and conditional text generation.



---

## 2. Hallucinations & Evaluation Metrics

### Question 4: What is hallucination in LLMs? Why does it happen, and what parameters influence it?

**Answer:**

* **Definition:** Hallucination occurs when an LLM generates plausible-sounding text that is factually incorrect, ungrounded, or completely fabricated relative to real-world knowledge or provided context.
* **Root Causes:**
* Probabilistic nature of next-token prediction without internal truth verification mechanisms.
* Out-of-distribution prompts, noisy training data, or missing context in the prompt payload.


* **Influencing Parameters:** High temperature, high Top-P, and long unstructured prompt sequences elevate hallucination risks.

---

### Question 5: How do you evaluate hallucination? What metrics are used?

**Answer:**

* **Faithfulness / Groundedness Metric:** Evaluates what fraction of claims made in the generated answer can be directly inferred from the retrieved reference context.
* **Context Recall & Precision:** Measures whether the necessary source text was retrieved and if irrelevant noise was excluded.
* **Evaluation Frameworks:** Frameworks like **RAGAS** or **DeepEval** utilize G-Eval (LLM-as-a-Judge) patterns alongside semantic entailment classifiers (NLI models) to systematically score hallucination severity before production release.

---

## 3. RAG Architecture, Vector DBs & Deployment

### Question 6: Explain the end-to-end RAG system design and pipeline.

**Answer:**

1. **Ingestion:** Document parsing, text extraction, semantic chunking, embedding generation via vector encoders.
2. **Indexing:** Storing vector representations and metadata in a vector database (e.g., FAISS, Pinecone).
3. **Retrieval:** User query embedding, approximate nearest neighbor (ANN) vector search, metadata filtering, optional reranking.
4. **Synthesis:** System prompt construction (injecting retrieved chunks), LLM inference, guardrail validation, and response streaming to the client.

---

### Question 7: How do you handle unrestricted vs. restricted data during retrieval in RAG systems?

**Answer:**

* **Pre-Retrieval Metadata Filtering:** Do not rely on prompt directives to restrict data access. Attach explicit security metadata tags (`role`, `department_id`, `access_level`) to vector chunks during ingestion.
* **RBAC Vector Scoping:** Scope the vector search query execution directly to match the authenticated user's access rights. Unrestricted data is accessible globally; restricted data chunks are returned *only* if the user's validated token matches chunk security tags.

---

### Question 8: What is FAISS, and what is PEFT?

**Answer:**

* **FAISS (Facebook AI Similarity Search):** An open-source library optimized for efficient similarity search and clustering of dense vectors, offering GPU-accelerated Indexing (e.g., IVF-PQ, HNSW) for high-speed retrieval.
* **PEFT (Parameter-Efficient Fine-Tuning):** A set of techniques (e.g., **LoRA**, **QLoRA**) that fine-tunes large language models by updating only a small fraction of parameters (adapter matrices) while freezing base model weights, drastically reducing compute and VRAM requirements.

---

### Question 9: How would you host your LLM for latency optimization when handling high traffic spikes?

**Answer:**

1. **Specialized Serving Frameworks:** Deploy models using high-throughput inference engines like **vLLM** or **TGI (Text Generation Inference)** utilizing PagedAttention and continuous batching.
2. **Quantization:** Load models in 4-bit or 8-bit precision (AWQ / GPTQ) to reduce memory bandwidth bottlenecks.
3. **Autoscaling & Replicas:** Deploy model replicas behind an asynchronous load balancer with autoscaling triggered by GPU utilization and queue depth.
4. **Caching & Speculative Decoding:** Implement semantic response caching (Redis) and speculative decoding (using a draft smaller model to speed up generation).

---

## 4. Python Fundamentals for AI Backends

### Question 10: What is the difference between Decorators and Generators in Python? How do Generators relate to streaming LLM responses?

**Answer:**

* **Decorator:** A higher-order function that takes another function as an argument, extends its behavior without explicitly modifying it, and returns a new function (e.g., `@app.get()`, `@time_it`).
* **Generator:** A function that produces a sequence of values over time using the `yield` keyword instead of `return`, maintaining its internal state between execution calls.
* **LLM Streaming Connection:** When an LLM API streams tokens back chunk-by-chunk (Time-To-First-Token optimization), Python backend services use **generators** (or async generators) to yield server-sent events (SSE) continuously to the frontend interface without loading the entire response into memory at once.

---

### Question 11: How do you handle and process large quantities of data in Python without encountering Out-Of-Memory (OOM) errors?

**Answer:**

1. **Iterative Data Streaming:** Use generators and file streaming streams (`ijson`, `pandas` chunking via `chunksize`) to process data sequentially line-by-line or batch-by-batch instead of loading entire datasets into memory.
2. **Memory-Efficient Data Structures:** Utilize `numpy` arrays, Polars, or PyArrow backends instead of native high-overhead Python objects.
3. **Distributed Processing:** Scale out large ETL tasks using distributed processing engines like PySpark or Dask when single-node memory capacity is exceeded.

---

## 5. Chunking Strategies & Model Context Protocols

### Question 12: What are the different types of chunking strategies, and what parameters do you pass during chunking?

**Answer:**

* **Strategies:**
* **Fixed-Size Chunking:** Splits text by character/token count with a fixed overlap window.
* **Recursive Chunking:** Splits hierarchy-wise by natural boundaries (paragraphs, sentences, punctuation) to keep semantic thoughts intact.
* **Semantic Chunking:** Computes embedding distances between adjacent sentences and splits text when semantic divergence exceeds a set threshold.


* **Key Parameters:** `chunk_size` (token/character limit), `chunk_overlap` (boundary continuity window), `separators` (delimiter priority order), and `length_function` (`len` vs. token counters).

---

### Question 13: What is Function Calling in an LLM, and when does the model trigger it?

**Answer:**

* **Function Calling:** An LLM capability where the model evaluates a user query against provided JSON schemas of external tools/functions, detects when external data or actions are required, and outputs structured JSON arguments matching the tool's parameter signature rather than plain text.
* **Triggering:** Triggered automatically when the user prompt demands external information (e.g., checking order status, querying a database) or side-effect actions that the LLM's static weights cannot resolve internally.

---

### Question 14: What is MCP (Model Context Protocol), and why was it introduced?

**Answer:**

* **Model Context Protocol (MCP):** An open standard designed to standardize connections between AI applications (LLM hosts) and external tools, databases, and context providers.
* **Why Introduced:** Eliminates the fragmentation of writing custom, proprietary API integration wrappers for every distinct tool and model provider by establishing a uniform server-client protocol interface for tool discovery, resource management, and prompt handling.
