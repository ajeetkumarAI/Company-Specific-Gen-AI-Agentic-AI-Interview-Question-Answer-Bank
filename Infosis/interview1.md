# Infosys Gen AI Interview Experience (2026-2027) — Complete Transcribed Questions & Answers

---

## 1. Introduction, Project Discussion & Agentic Workflows

### Question 1: Tell me about yourself and what exactly are you working on currently?

**Answer:**

* **Project Overview:** Discussed hands-on experience building production-grade Generative AI applications, focusing on architectural design, RAG implementations, and agentic workflows.
* **Individual Contribution:** Explained personal ownership in designing the orchestration layer, integrating external tools, and managing backend execution flows.

---

### Question 2: You mentioned Agentic AI. What exactly is Agentic AI in your application?

**Answer:**

* **Definition & Core Logic:** An AI agent goes beyond simple single-step text generation by using an LLM as a central reasoning engine to evaluate inputs, determine required actions, invoke external tools or APIs, receive feedback, and continue execution loops autonomously until the task is complete.

---

### Question 3: Who decides which tool to call, and what if the model selects the wrong tool?

**Answer:**

* **Tool Orchestration:** The model selects appropriate tools based on the task prompt and available tool descriptions, while the application's orchestration layer strictly controls which tools are exposed and what permissions they hold.
* **Handling Incorrect Tool Selection (Production Scenarios):**
* Use clear, explicit **tool descriptions** and parameter schemas (e.g., Pydantic).
* Implement **restricted tool access** and strict input/output validation.
* Build robust **fallback handling** and loop limits so the agent fails gracefully or asks for clarification rather than cascading errors.



---

## 2. Model Context Protocol (MCP)

### Question 4: Have you worked with MCP? Explain MCP in your own words.

**Answer:**

* **Definition:** The **Model Context Protocol (MCP)** is an open standard that provides a standardized way for AI applications (model hosts) to connect with external tools, data sources, and context providers.
* **Components:** MCP servers expose capabilities like tools, resources, and prompts, which an MCP host can dynamically discover and connect to.

---

### Question 5: What is the difference between normal function calling and MCP?

**Answer:**

* **Traditional Function Calling:** Implemented directly and custom-built between a specific monolithic application and its hardcoded functions.
* **Model Context Protocol (MCP):** Provides a universal, standardized protocol layer. Instead of custom point-to-point integrations for every tool and application, MCP decouples the client from external capabilities, allowing any MCP-compatible host to dynamically discover and plug into various MCP servers seamlessly.

---

## 3. RAG vs. Fine-Tuning

### Question 6: When would you choose fine-tuning over RAG, and can you use both?

**Answer:**

* **RAG vs. Fine-Tuning:** They solve different problems and are not mutually exclusive competitors.
* **Use RAG:** When the model needs access to external, private, and frequently changing enterprise knowledge without retraining.
* **Use Fine-Tuning:** When you need to modify the model's fundamental behavior, tone, style, formatting rules, or specialized domain-specific task capabilities.


* **Combining Both:** Yes, you can use both simultaneously—for example, deploying a fine-tuned model optimized for domain tone that is also grounded via real-time context retrieved through a RAG pipeline.

---

## 4. LLM & RAG Evaluation

### Question 7: How do you know your Gen AI application is actually getting better? (Offline vs. Production Evaluation)

**Answer:**

* **Multi-Level Evaluation:** Relying solely on user satisfaction ("users are happy") is insufficient. Evaluation must be systematic:
* **Retrieval Layer:** Separately evaluate the retriever using precision@K, recall@K, and ranking metrics.
* **Generation Layer:** Evaluate generated outputs against retrieved context using metrics like faithfulness, relevance, and correctness.


* **Handling Discrepancies (Offline Benchmarks vs. Production Realities):**
* If offline evaluation scores look great but users still complain, it means benchmark datasets do not reflect real-world user distribution.
* **Resolution:** Capture actual production failure traces, difficult queries, edge cases, and user feedback, and continuously feed those real-world examples back into the evaluation test suite.



---

## 5. Python Engineering & Backend Fundamentals

### Question 8: Why would you use async programming in an AI application? Suppose your application has to call three independent APIs—would you call them sequentially or concurrently?

**Answer:**

* **Async Programming:** Essential in AI backends to handle I/O-bound operations efficiently without blocking the event loop while waiting on external network calls.
* **Concurrent Execution:** For three independent API calls (e.g., fetching user data, querying a vector DB, and checking order status), use **concurrent/asynchronous execution** (e.g., `asyncio.gather`) instead of sequential calls to drastically reduce overall latency and waiting time.

---

### Question 9: What is the difference between concurrency and parallelism, and where does the Python GIL become relevant?

**Answer:**

* **Concurrency vs. Parallelism:** Concurrency is about dealing with lots of things at once (task switching, ideal for I/O-bound tasks), while parallelism is about doing lots of things at the exact same instant (true multi-core execution, required for CPU-bound math like embeddings or text chunking).
* **Python GIL (Global Interpreter Lock):** A mutex in CPython that prevents multiple native threads from executing Python bytecodes at once. It becomes relevant in multi-threaded applications because it restricts CPU-bound parallel processing, necessitating the use of `multiprocessing` for heavy CPU workloads.

---

## 6. Advanced Agentic Scenarios & Production Guardrails

### Question 10: Scenario: Imagine your agent has access to 10 tools. The user asks a simple question, but the agent starts calling 5 different tools. How would you control this?

**Answer:**

* **Tool Selection Controls:** Having access to tools does not mean the agent should use all of them blindly.
* Implement strict **tool selection constraints**, routing logic, and system prompt instructions that guide the agent on when a tool is genuinely necessary.
* Use **deterministic routing** for known, repetitive workflows to bypass unnecessary LLM reasoning steps.
* Set strict **iteration limits** and timeout thresholds on tool calls to prevent runaway loops.



---

### Question 11: Would you completely trust the LLM to perform a sensitive action like deleting a customer record?

**Answer:**

* **No.** Unrestricted LLM power over production systems introduces severe operational and security risks.
* **Production Safeguards:** Enforce application-level authorization, role-based access control (RBAC), and mandatory **Human-in-the-Loop (HITL) approvals** before any state-changing, destructive, or sensitive action (like data deletion or financial transactions) is executed.
