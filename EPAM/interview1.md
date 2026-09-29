# EPAM Systems — Senior AI Engineer Interview Guide (~25 LPA Level)

This document provides a comprehensive, production-grade interview guide for the **Senior AI Engineer / LLM Engineer** role at **EPAM Systems**, based directly on real interview transcripts. EPAM's evaluation heavily emphasizes end-to-end SDLC breakdown, hands-on production engineering, agentic state management, concurrency control, and system design.

---

## 1. Candidate Background & Interview Process Summary

* **Role & Company:** Senior AI / LLM Engineer at **EPAM Systems** (Target Compensation Band: ~25 LPA).
* **Focus Areas:** End-to-end System Design, Full SDLC Execution, Multi-Agent Orchestration (LangGraph), Production Incident Data Management, Fast-API Integration, and LLM/Agentic Reliability.
* **Round Structure:**
  1. **Screening Round (Phone):** Career background, previous project architectures, ownership scope, and SDLC responsibilities.
  2. **Technical Round 1 (System Design & Architecture - 45-60 Mins):** Oral whiteboarding, async concurrency, multi-agent fault isolation, state management, Fast-API integration, MCP, and fundamental NLP/LLM mechanics.
  3. **Technical Round 2 (Hands-On Coding & Implementation):** Live algorithmic coding, system-level Python scripting, and data structure manipulation.
  4. **Managerial & Offer Discussion:** Final evaluation of overall performance, client consulting readiness, and compensation mapping.

---

## 2. Multi-Agent Systems, Concurrency & Resilience

### Question 1: How would you manage incident data efficiently in a database to support fast execution and low-latency RAG retrieval?
**Answer:**
* **Database Partitioning:** Partition incident tables by **Time Window** (e.g., monthly/weekly) and **Severity** (e.g., `Critical`, `High`, `Low`) to isolate heavy read/write volumes and avoid full-table scans.
* **Indexing Strategy:** Create composite B-Tree indexes on heavily queried filtering columns, specifically `(incident_id, timestamp, severity)`.
* **Hybrid RAG Storage:**
  * Keep structured operational data (status, assigned team, timestamps) in a relational store (PostgreSQL) or document store with partitioning.
  * Offload raw, unstructured incident logs and post-mortems to a vector database (or `pgvector`) with metadata payload filters (`incident_id`, `created_at`) to enable fast, targeted semantic retrieval during automated RAG incident analysis.

---

### Question 2: How would you execute three independent AI Agents in parallel, and how do you isolate failures so one agent's crash does not affect the others?


```

```
                    [ User Request / Event ]
                               │
                               ▼
                  [ async Task Orchestrator ]
                    (asyncio.gather / TaskGroup)
                               │
     ┌─────────────────────────┼─────────────────────────┐
     ▼                         ▼                         ▼

```

[ Agent A Task ]          [ Agent B Task ]          [ Agent C Task ]
(Wrapped in try/except)   (Wrapped in try/except)   (Wrapped in try/except)
│                         │                         │
▼                         ▼                         ▼
[ Success / Error ]       [ Success / Error ]       [ Success / Error ]
│                         │                         │
└─────────────────────────┼─────────────────────────┘
│
▼
[ Aggregate Results & Fallbacks ]

```

**Answer:**
* **Asynchronous Execution:** Use Python's `asyncio.gather(*tasks, return_exceptions=True)` or `asyncio.TaskGroup()` to spawn and run the agents concurrently on an async event loop, significantly reducing overall execution time compared to sequential calls.
* **Fault Isolation:** Wrap each agent's execution logic in an isolated `try...except` block or execute them via safe worker tasks. 
* **Handling `return_exceptions=True`:** Setting `return_exceptions=True` ensures that if Agent B throws an unhandled exception or API timeout, `asyncio.gather` does not cancel Agent A and Agent C. Instead, it captures the Exception object for Agent B while returning valid outputs for Agents A and C.

```python
import asyncio
import logging
from typing import Dict, Any, List, Union

async def run_agent_a() -> Dict[str, Any]:
    # Simulating LLM call / Tool execution
    await asyncio.sleep(0.5)
    return {"agent": "Agent_A", "status": "success", "data": "Analysis complete"}

async def run_agent_b() -> Dict[str, Any]:
    await asyncio.sleep(0.3)
    # Simulating a failure (e.g., API Rate Limit / Timeout)
    raise TimeoutError("LLM Provider timed out during agent processing")

async def run_agent_c() -> Dict[str, Any]:
    await asyncio.sleep(0.4)
    return {"agent": "Agent_C", "status": "success", "data": "Routing verified"}

async def orchestrate_parallel_agents() -> List[Dict[str, Any]]:
    """
    Executes multiple agents concurrently while isolating individual failures.
    """
    tasks = [
        run_agent_a(),
        run_agent_b(),
        run_agent_c()
    ]
    
    # return_exceptions=True prevents one failing task from cancelling others
    results: List[Union[Dict[str, Any], Exception]] = await asyncio.gather(*tasks, return_exceptions=True)
    
    processed_responses = []
    for idx, result in enumerate(results):
        if isinstance(result, Exception):
            logging.error(f"Task {idx} failed with error: {result}")
            processed_responses.append({
                "agent_index": idx,
                "status": "failed",
                "error": str(result),
                "fallback_data": None
            })
        else:
            processed_responses.append(result)
            
    return processed_responses

# Run event loop
if __name__ == "__main__":
    output = asyncio.run(orchestrate_parallel_agents())
    print(output)

```

---

### Question 3: How do you design an automated retry mechanism with backoff for Agent execution?

**Answer:**

* Use an explicit **Retry Counter** alongside **Exponential Backoff with Jitter** to prevent overwhelming downstream LLM providers or external API tools during rate limiting ($429$) or server errors ($5xx$).
* Stop retrying once `max_retries` is reached, and return a structured fallback response or route to an error state.

```python
import asyncio
import random
from typing import Callable, Any

async def execute_with_retry(
    agent_func: Callable[[], Any], 
    max_retries: int = 3, 
    initial_delay: float = 1.0
) -> Any:
    """
    Executes an async agent function with exponential backoff and jitter.
    """
    delay = initial_delay
    for attempt in range(1, max_retries + 1):
        try:
            return await agent_func()
        except Exception as e:
            if attempt == max_retries:
                print(f"[FINAL FAILURE] Max retries ({max_retries}) reached. Error: {e}")
                return {"status": "failed", "reason": str(e)}
            
            # Exponential delay calculation + random jitter
            jitter = random.uniform(0, 0.5)
            sleep_time = delay + jitter
            print(f"[RETRY {attempt}/{max_retries}] Sleeping for {sleep_time:.2f}s due to: {e}")
            await asyncio.sleep(sleep_time)
            delay *= 2  # Double the backoff interval

```

---

## 3. LangGraph, Chains vs. Agents, and Fast-API Integration

### Question 4: What is the primary difference between a Chain and an Agent?

**Answer:**

* **Chain (e.g., LCEL / Sequential Pipeline):** A hardcoded, deterministic execution graph where step sequence is predefined (`Step A -> Step B -> Step C`). The LLM is used strictly for data transformation or extraction at specific nodes.
* **Agent:** An autonomous control loop driven dynamically by an LLM acting as a reasoning engine. The agent evaluates the user input, decides which tools to call, inspects tool outputs, and dynamically determines the next step until the objective is satisfied.

---

### Question 5: How is State managed in LangGraph?

**Answer:**

* **State Schema via `TypedDict` or Pydantic:** LangGraph manages state by defining a centralized schema (a Python `TypedDict` or `BaseModel`) that represents the memory payload accessible by all graph nodes.
* **Node Updates & Reducers:** Each node in a LangGraph graph receives the current state, executes its processing logic, and returns a dictionary with updated fields.
* **State Progression:** Channels within the state can use reducer functions (e.g., `operator.add` for appending messages to a list) to update state deterministically as execution transitions between nodes.

```python
from typing import Annotated, TypedDict
import operator

# Defining LangGraph State using TypedDict and Reducers
class AgentGraphState(TypedDict):
    # 'operator.add' ensures new messages are appended rather than overwritten
    messages: Annotated[list[str], operator.add]
    current_agent: str
    is_complete: bool
    retry_count: int

```

---

### Question 6: Why is Fast-API preferred for Gen AI agent backends, and how does Pydantic fit in?

**Answer:**

* **Fast-API:** Provides high-throughput, native asynchronous (`async/await`) HTTP support built on Starlette and Pydantic. It enables non-blocking I/O operations necessary when streaming tokens or handling long-running LLM tool calls.
* **Pydantic Validation:** Fast-API uses Pydantic models to automatically validate incoming JSON request payloads and enforce typed response schemas. For LLM backends, Pydantic is used to enforce **Structured Outputs** (e.g., tool schemas, function calling signatures) directly from the model.

---

## 4. MCP, Claude Integration, and LLM Hyperparameters

### Question 7: What is MCP (Model Context Protocol), and how does its integration work?

**Answer:**

* **Model Context Protocol (MCP):** An open standard designed by Anthropic that standardizes how LLM clients securely connect to local or remote context providers (databases, search tools, enterprise applications).
* **Integration Mechanics:** Operates via a **Client-Server Architecture**. An MCP Server exposes capability definitions (tools, prompts, resources) using a standardized JSON-RPC protocol over standard I/O (`stdio`) or Server-Sent Events (SSE). The LLM Client (e.g., Claude Desktop, custom agent) connects to the server, discovers exposed capabilities, and invokes them cleanly without requiring custom custom API wrappers for every separate data source.

---

### Question 8: Explain the differences between Temperature, Top-P, and Top-K.

**Answer:**

* **Temperature:** Scales the unnormalized logit values before applying the Softmax function.
* $T \to 0$ (Greedy Decoding): Makes the model output deterministic by always choosing the highest-probability token.
* High $T$: Flattens the probability distribution, encouraging variety and creativity.


* **Top-K:** Constrains selection strictly to the top $K$ most probable next tokens, zeroing out probabilities for all tokens ranked outside $K$.
* **Top-P (Nucleus Sampling):** Selects tokens dynamically from the top candidate pool whose cumulative probability sum reaches or exceeds threshold $P$ (e.g., $P = 0.90$). The candidate set expands or contracts based on the model's confidence across different context points.

---

### Question 9: How do LLMs understand words with multiple meanings (e.g., "bank" of a river vs. financial "bank")?

**Answer:**

* **Self-Attention Mechanism:** The Transformer's self-attention mechanism evaluates pairwise relationships between every token in the input sequence simultaneously.
* **Contextual Vector Representation:** Rather than using static word lookup embeddings (like Word2Vec), the self-attention heads compute dynamic Query ($Q$), Key ($K$), and Value ($V$) transformations. The representation of the word "bank" changes dynamically based on attention weights assigned to surrounding context tokens (e.g., "river", "water", "mud" vs. "money", "deposit", "interest").

---

### Question 10: How do you detect and handle hallucinations in an Agentic system?

**Answer:**

1. **Context Grounding Audit:** Verify generated claims directly against retrieved source documents using an entailment classifier or automated evaluation suite (RAGAS / DeepEval).
2. **ReAct Reflection & Verification Loops:** Program an explicitly defined "Verification Node" in LangGraph that evaluates tool outputs against generated text before finalizing output execution.
3. **Strict System Constraints:** Use deterministic system prompts mandating refusal ("I do not know") if facts are absent from the context.
4. **Self-Correction Retry:** If a hallucination is detected at an intermediary step, trigger a retry loop passing the error state back to the agent to regenerate the response with strict constraints.

---

## 5. Coding Challenge & Production Implementation

### Problem: Find the Nearest Palindromic Number

* **Task:** Given an integer string `n`, find the closest palindromic integer (excluding `n` itself). Distance is measured by absolute numerical difference. If there is a tie, return the smaller numerical value.

```python
def nearest_palindromic(n: str) -> str:
    """
    Calculates the nearest palindromic number to string input n.
    Handles edge cases including scale changes, ties, and boundary conditions.
    """
    length = len(n)
    num = int(n)
    
    # Base candidates for scale boundaries (e.g., 100 -> 99, 99 -> 101)
    candidates = {
        str(10**(length - 1) - 1),  # Lower scale bound (e.g., 99...9)
        str(10**length + 1)         # Upper scale bound (e.g., 100...01)
    }
    
    # Extract prefix
    prefix_len = (length + 1) // 2
    prefix = int(n[:prefix_len])
    
    # Generate candidates by modifying prefix (-1, 0, +1)
    for i in (-1, 0, 1):
        new_prefix = str(prefix + i)
        if length % 2 == 0:
            # Even length: mirror entire prefix
            candidate = new_prefix + new_prefix[::-1]
        else:
            # Odd length: mirror prefix excluding middle digit
            candidate = new_prefix + new_prefix[:-1][::-1]
        candidates.add(candidate)
        
    # Remove original number if present
    candidates.discard(n)
    
    # Select candidate with minimum absolute difference
    # Tie-breaker: choose smallest numerical value
    best_candidate = None
    min_diff = float('inf')
    
    for cand in candidates:
        cand_num = int(cand)
        diff = abs(cand_num - num)
        
        if diff < min_diff:
            min_diff = diff
            best_candidate = cand_num
        elif diff == min_diff:
            best_candidate = min(best_candidate, cand_num)
            
    return str(best_candidate)

# Test Execution
if __name__ == "__main__":
    test_cases = ["123", "1", "10", "88", "1001"]
    for tc in test_cases:
        print(f"Input: {tc} -> Nearest Palindrome: {nearest_palindromic(tc)}")

```

The comparison between **Claude Code** and **Claude CoWork** was included in the guide under **Section 4, Question 7**.

An expanded breakdown highlights the specific differences:

---

### Claude Code vs. Claude CoWork

```
                         [ Shared Underlying Model ]
                      (Claude 3.7 Sonnet / Opus Engine)
                                     │
           ┌─────────────────────────┴─────────────────────────┐
           ▼                                                   ▼
  [ Claude Code ]                                     [ Claude CoWork ]
  • Developer-focused                                 • Non-technical/Operations-focused
  • CLI / Terminal / IDE                             • Desktop App GUI / Sandboxed VM
  • Codebases, Git, CI/CD, Shell                     • Documents, Spreadsheets, Research
  • MCP via Config / Code Hooks                       • Connector Marketplace / UI Plugins

```

| Dimension | **Claude Code** | **Claude CoWork** |
| --- | --- | --- |
| **Target Audience** | Software Engineers, LLM/AI Engineers, DevOps | Non-technical knowledge workers, Operations, Legal, Finance |
| **Primary Interface** | Command-Line Interface (CLI / Terminal) & IDE extension | GUI (Integrated inside the Claude Desktop App) |
| **Execution Surface** | Direct filesystem/shell access within project workspace | Sandboxed virtual workspace / isolated environment |
| **Primary Deliverables** | Pull Requests, Git diffs, feature implementation, refactors, unit tests | Formatted reports, spreadsheets, contract data extraction, presentation slides |
| **Tooling & Integrations** | Config-file-driven MCP servers, system hooks, terminal tools | Plug-and-play visual Connector Marketplace (Salesforce, Slack, Notion) |
| **Workflow Model** | Plan mode, strict human-in-the-loop permission gates, diff review | Automated background task execution and scheduled desktop workflows |

---

### Key Interview Response Summary

> *"While both run on the same agentic loops and underlying Claude models, **Claude Code** is tailored for software engineering workflows (terminal execution, multi-file code diffs, Git operations, and direct shell tool access). **Claude CoWork**, on the other hand, targets general business productivity—running in an isolated, sandboxed desktop app to handle document processing, multi-source research, and office task automation using a click-and-connect marketplace rather than raw config files."*

---

## 6. Preparation Takeaways for EPAM Senior AI Engineer Roles

1. **Master Asynchronous Concurrency:** Be ready to demonstrate `asyncio.gather`, task isolation (`return_exceptions=True`), exponential backoff, and state-preserving error handling live during system design discussions.
2. **State Management Expertise:** Know how state propagates inside **LangGraph** (`TypedDict`, state channels, state reducers) and how to manage state transitions across multi-agent workflows.
3. **Bridge System Design with Classical Computer Science:** Be prepared to transition smoothly between system design concepts (Fast-API, partitioning, database indexing) and fundamental algorithms/data manipulation tasks.

```

```
