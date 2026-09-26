# TCS Generative AI Engineer Interview (2026-2027) - Complete Question & Answer Set

---

## 1. Project Deep-Dive & Core Architecture

### Q1: Explain your Generative AI project architecture from end to end.
**Answer:**
* **Use Case:** Enterprise Knowledge Base / Policy Assistant that answers user queries based on internal domain documents.
* **Architecture Flow:**
  1. **Ingestion & Preprocessing:** Parsing PDFs, Markdown, or Word documents; cleaning noise/metadata.
  2. **Chunking & Embedding:** Splitting documents into manageable text chunks and generating dense vector embeddings using models like `text-embedding-3-small` or `bge-large-en`.
  3. **Vector Database:** Storing embeddings alongside document metadata (e.g., page number, file name, access permissions) in a vector store like Pinecone, Qdrant, or FAISS.
  4. **Retrieval Phase:** Generating query embeddings, executing dense semantic retrieval (Top-K similarity search), and applying reranking (e.g., Cohere Rerank) to filter the top 3–5 chunks.
  5. **Generation Phase:** Injecting retrieved context into a structured prompt, passing it to the generator LLM (e.g., Claude 3.5 Sonnet or GPT-4o), and enforcing response schemas or citations.

### Q2: Why did you choose RAG over using a standalone LLM or Fine-Tuning?
**Answer:**
* **Compared to Standalone LLM:** Standalone base models lack access to private, real-time, or domain-specific company data and are prone to hallucinations when asked about internal documents.
* **Compared to Fine-Tuning:**
  * **Dynamic Knowledge Updating:** RAG updates its knowledge base instantaneously whenever new files are uploaded, whereas Fine-Tuning requires costly and time-consuming GPU retraining runs.
  * **Verifiability & Citations:** RAG allows strict source attribution by citing chunk metadata; fine-tuned model weights store knowledge implicitly without verifiable source references.
  * **Cost Efficiency & Access Control:** Fine-tuning base weights cannot enforce user-level security permissions, while RAG can easily enforce Role-Based Access Control (RBAC) metadata filters during vector retrieval.

---

## 2. RAG Pipeline & Retrieval Optimization

### Q3: What chunking strategy did you use, and why?
**Answer:**
* **Fixed-size Chunking (with overlap):** e.g., 500-token chunks with a 50-token overlap. Simple, but can break semantic sentences across boundaries.
* **Recursive Character Text Chunking:** Splits text hierarchically using double line breaks (`\n\n`), single line breaks (`\n`), spaces, and characters. This preserves logical paragraphs and complete thought structures.
* **Semantic Chunking:** Calculates embedding distances between adjacent sentences and creates split boundaries when vector similarity drops below a predefined threshold, ensuring each chunk contains a single coherent concept.
* **Parent-Child / Hierarchical Chunking:** Uses small child chunks (e.g., 100–150 tokens) for precise vector distance calculation during search, but returns the larger parent block (e.g., 1000 tokens) to the generator model to provide rich context.

### Q4: How do you evaluate retrieval quality? Explain Precision@K, Recall@K, and MRR.
**Answer:**
* **Precision@K:** Measures the proportion of relevant chunks within the top $K$ retrieved chunks:
  $$\text{Precision@K} = \frac{\text{Number of Relevant Chunks in Top } K}{K}$$
* **Recall@K:** Measures the fraction of all truly relevant chunks in the database that were successfully retrieved in the top $K$:
  $$\text{Recall@K} = \frac{\text{Number of Relevant Chunks in Top } K}{\text{Total Number of Relevant Chunks in Dataset}}$$
* **Mean Reciprocal Rank (MRR):** Evaluates the position of the *first* relevant result across multiple test queries:
  $$\text{MRR} = \frac{1}{\vert{}Q\vert{}} \sum_{i=1}^{\vert{}Q\vert{}} \frac{1}{\text{rank}_i}$$
  where $\text{rank}_i$ is the 1-based position of the first relevant chunk returned for query $i$.

---

## 3. LLM Parameters, Prompts & Hallucination Mitigation

### Q5: What is the difference between System Prompt and User Prompt?
**Answer:**
* **System Prompt:** Sets the global behavior, role boundaries, tone, constraints, and safety guardrails for the model (e.g., *"You are an internal corporate support assistant. Only answer queries using the provided context."*). It carries higher structural priority in model instructions.
* **User Prompt:** The dynamic, variable query or input provided by the end-user during a specific conversation turn (e.g., *"What is our company's parental leave policy?"*).

### Q6: How do LLM parameters like Temperature, Top-P, and Top-K affect generation?
**Answer:**
* **Temperature:** Controls output randomness by flattening or steepening the Softmax probability distribution over tokens.
  * **Low Temperature ($0.0 - 0.2$):** Makes outputs deterministic and factual (ideal for RAG/code).
  * **High Temperature ($0.7 - 1.0$):** Increases output creativity and variability.
* **Top-P (Nucleus Sampling):** Selects tokens from the smallest cumulative probability set whose sum equals $P$ (e.g., Top-P = $0.9$ samples from tokens making up the top $90\%$ probability mass).
* **Top-K Sampling:** Restricts token choices strictly to the top $K$ most probable next tokens, discarding all others.

### Q7: How do you detect and reduce LLM hallucinations in a production system?
**Answer:**
1. **RAG Grounding:** Restrict the system prompt to explicitly state: *"Answer strictly based on the context. If the answer is not contained within the context, state 'I do not have enough information to answer.'"*
2. **Self-Correction & Reranking:** Run a lightweight secondary LLM step or evaluator to verify if every assertion in the output is supported by the context before returning the response.
3. **Guardrails & Structured Outputs:** Enforce Pydantic output parsing or frameworks like NeMo Guardrails to reject ungrounded generations.
4. **Lower Temperature:** Set temperature to $0.0$ to minimize creative divergence.

---

## 4. Agentic AI & Tool Integration

### Q8: What is Agentic AI, and how does it differ from standard Generative AI?
**Answer:**
* **Standard Generative AI:** Single-turn input-to-output mapping (e.g., drafting an email or generating a single response from a prompt).
* **Agentic AI:** Autonomous systems capable of multi-step planning, tool selection, environment interaction, and self-correction to achieve complex end goals without step-by-step human intervention.

### Q9: How would you design an AI Agent with external tools?
**Answer:**
* Uses the **ReAct (Reasoning + Acting)** framework:
  1. **Thought:** The agent breaks down the problem and determines which tool is needed.
  2. **Action:** The agent outputs a structured tool invocation request (e.g., calling a REST API or executing an SQL query).
  3. **Observation:** The environment runs the tool and returns the response back into the context window.
  4. **Iteration:** The agent evaluates the output and decides the next action or presents the final answer.

### Q10: How do you prevent AI Agents from getting trapped in infinite execution loops?
**Answer:**
1. **Max Iteration Limits:** Enforce strict execution step counters (`max_iterations = 5`).
2. **State & Duplicate Action Tracking:** Maintain an action log; if the agent attempts the exact same tool call with identical parameters twice in a row, interrupt execution.
3. **Fallback & Graceful Timeout Handlers:** Implement system timeouts that force the agent into a human-in-the-loop or default error message state if task completion takes too long.

---

## 5. Python Core & System Performance

### Q11: What is the difference between Multithreading and Multiprocessing in Python?
**Answer:**
* **Multithreading (`threading` module):**
  * Multiple threads run within the same process memory space.
  * Subject to Python's Global Interpreter Lock (GIL), meaning threads cannot run CPU-bound code in parallel across multiple CPU cores.
  * **Best for:** I/O-bound tasks (e.g., asynchronous API calls, fetching data from vector DBs).
* **Multiprocessing (`multiprocessing` module):**
  * Creates separate process instances, each with its own memory space and Python interpreter instance.
  * Bypasses the GIL, enabling true parallel execution across multi-core CPUs.
  * **Best for:** CPU-bound tasks (e.g., tokenization, chunking, data processing, heavy local model inference).

### Q12: What is the Python Global Interpreter Lock (GIL)?
**Answer:**
* The **GIL** is a mutex (mutual exclusion lock) used by the CPython interpreter to ensure that only one thread executes Python bytecode at any given moment.
* It prevents race conditions and ensures memory safety in CPython's reference-counting garbage collection system, but limits Python multithreading from leveraging multi-core parallel computing for heavy mathematical or CPU computations.

---

## 6. Production Operations, Monitoring & Debugging

### Q13: How do you debug high latency issues in a RAG pipeline?
**Answer:**
1. **Add End-to-End Tracing:** Integrate observability platforms like LangSmith, Arize Phoenix, or OpenTelemetry to measure latency at each individual pipeline step:
   $$\text{Total Latency} = T_{\text{embedding}} + T_{\text{vector\_search}} + T_{\text{reranking}} + T_{\text{LLM\_TTFT}} + T_{\text{generation}}$$
2. **Identify the Bottleneck:**
   * **If Vector Retrieval is slow:** Optimize database indexing (e.g., switch from exact search to approximate nearest neighbor HNSW), or cache frequent embedding results.
   * **If Generation is slow:** Enable streaming responses (Time to First Token - TTFT), use smaller/quantized LLMs, or adopt prompt caching strategies.

### Q14: What metrics should you continuously monitor for a production Gen AI application?
**Answer:**
* **System Metrics:** Latency (TTFT, Total Response Time), API Error Rates, GPU Memory/Compute Utilization, Token Throughput (tokens/sec).
* **Quality & RAG Metrics:** Context Relevance, Answer Groundedness (Faithfulness), User Feedback (Thumbs Up / Thumbs Down ratios).
* **Operational & Cost Metrics:** Daily Token Consumption (Input vs. Output tokens), API Cost Per User Query, Guardrail Trigger Rates (blocked prompt injections / toxic inputs).
