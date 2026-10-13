# TCS — Generative AI Engineer Technical Interview Guide (Coding & Production Concurrency Round)

This document provides a comprehensive, production-grade guide based on the live transcript of a **Generative AI Engineer Technical Coding Round (Round 2)** at **TCS** for candidates with 3–7 years of experience. 

Unlike theoretical Q&A sessions or LeetCode algorithmic puzzles, this interview evaluates **real-world backend engineering competence**, focusing on streaming Server-Sent Events (SSE), incremental UTF-8 decoding, concurrency rate-limiting with semaphores, exponential backoff with jitter, and distinction between concurrency and rate-limiting limits.

---

## 1. Candidate Background & Interview Overview

* **Company & Role:** Generative AI Engineer at **TCS** (Round 2 — Technical Coding).
* **Format:** 60-Minute Live Coding Session via Shared Online Editor (No internet search access).
* **Evaluation Core:** 
  1. Live code generation without LLM assistance.
  2. Continuous verbalization of thought process and architectural trade-offs.
  3. Relentless follow-up questioning after every code block, targeting production failure modes (network cuts, broken multi-byte characters, thundering herd risks, and token bucket rate limits).

---

## 2. Coding Problem 1: Streaming Server-Sent Events (SSE) Client & Incremental UTF-8 Decoding

### Base Problem Statement:
Write a mock streaming backend endpoint (similar to OpenAI's SSE format, where chunks arrive prefixed with `data: JSON\n` and terminate with `data: [DONE]\n`) and a corresponding client that reads incoming token streams, prints them as they arrive, and tracks **Time-To-First-Token (TTFT)** and **Tokens Per Second**.

---

### Candidate Code Implementation & Production Engineering

```python
import asyncio
import json
import time
import logging
from typing import AsyncGenerator, Dict, Any

logging.basicConfig(level=logging.INFO)

# --- 1. Mock Server Endpoint (Async Generator) ---
async def mock_openai_sse_server(prompt: str) -> AsyncGenerator[bytes, None]:
    """
    Simulates an LLM streaming endpoint emitting raw byte chunks, 
    intentionally splitting multi-byte UTF-8 characters across chunk boundaries to test resilience.
    """
    tokens = [
        "नमस्ते ",  # Hindi multi-byte characters
        "world! ",
        "Streaming ",
        "LLM ",
        "responses "
    ]
    
    for token in tokens:
        payload = {"choices": [{"delta": {"content": token}}]}
        sse_message = f"data: {json.dumps(payload)}\n\n".encode('utf-8')
        
        # Simulate network latency
        await asyncio.sleep(0.1)
        
        # Intentionally slice bytes awkwardly to test incremental decoding
        midpoint = len(sse_message) // 2
        yield sse_message[:midpoint]
        await asyncio.sleep(0.05)
        yield sse_message[midpoint:]
        
    yield b"data: [DONE]\n\n"


# --- 2. Production Resilient SSE Client ---
async def consume_llm_stream(prompt: str) -> None:
    """
    Consumes a streaming LLM response using an incremental UTF-8 decoder 
    and a line buffer to handle fragmented network chunks safely.
    """
    start_time = time.time()
    ttft = None
    token_count = 0
    
    buffer = bytearray()
    # Incremental decoder prevents UnicodeDecodeError when multi-byte characters are split across chunks
    decoder = json.JSONDecoder() # or codecs.getincrementaldecoder('utf-8')()
    utf8_decoder = codecs.getincrementaldecoder('utf-8')()

    try:
        async for raw_bytes in mock_openai_sse_server(prompt):
            # Decode chunks incrementally without throwing errors on incomplete multi-byte sequences
            decoded_chunk = utf8_decoder.decode(raw_bytes, final=False)
            buffer.extend(decoded_chunk.encode('utf-8'))
            
            # Process complete lines from the buffer
            while b'\n' in buffer:
                line_bytes, _, buffer = buffer.partition(b'\n')
                line = line_bytes.decode('utf-8', errors='ignore').strip()
                
                if not line:
                    continue
                
                if line == "data: [DONE]":
                    logging.info("[Stream Complete] Received termination signal.")
                    break
                    
                if line.startswith("data: "):
                    json_str = line[6:]
                    try:
                        data = json.loads(json_str)
                        content = data["choices"][0]["delta"].get("content", "")
                        
                        if content:
                            if ttft is None:
                                ttft = time.time() - start_time
                            token_count += 1
                            print(content, end="", flush=True)
                    except json.JSONDecodeError:
                        # Incomplete JSON fragment; push back or wait for next chunk
                        continue
                        
        total_time = time.time() - start_time
        tokens_per_sec = token_count / total_time if total_time > 0 else 0
        
        print(f"\n\n--- Metrics ---")
        print(f"Time-To-First-Token (TTFT): {ttft:.4f}s" if ttft else "TTFT: N/A")
        print(f"Throughput: {tokens_per_sec:.2f} tokens/sec")

    except Exception as e:
        logging.error(f"[Stream Interrupted] Error during consumption: {e}")
        # Implement reconnection or partial response state save here

```

---

### Follow-Up Questions & Production Edge Cases

#### 1. Interviewer Question: *"What happens if a network chunk cuts right in the middle of a line or a multi-byte UTF-8 character (like Hindi or Emojis)?"*

* **Candidate Answer:**
* Directly decoding raw byte slices using `.decode('utf-8')` throws a `UnicodeDecodeError` if a multi-byte sequence is severed midway.
* **Solution:** We use Python's `codecs.getincrementaldecoder('utf-8')()`, which retains incomplete byte fragments in an internal state buffer until the subsequent matching bytes arrive to form a valid Unicode character. Furthermore, a line-buffering mechanism (`buffer.partition(b'\n')`) ensures we only parse JSON events after encountering a full newline boundary.



#### 2. Interviewer Question: *"If the connection drops midway through a stream, how do you handle it?"*

* **Candidate Answer:**
* LLM streaming endpoints generally do not support resumable ranges (`Range: bytes=...`) like static file downloads.
* **Mitigation:** Catch the connection drop exception, retain whatever token text was successfully parsed and printed so far, and append a metadata flag or system warning (`[Response incomplete due to network timeout]`) before delivering the partial response to the user interface.



#### 3. Interviewer Question: *"How do you implement Timeouts for streaming responses?"*

* **Candidate Answer:**
* We use `asyncio.wait_for()`. In production, we maintain **two distinct timeouts**:
1. **Initial Connection / TTFT Timeout:** (e.g., max $2.0\text{s}$ for the first token to arrive).
2. **Inter-chunk Idle Timeout:** (e.g., max $5.0\text{s}$ pause between consecutive tokens to prevent hanging connections if the upstream LLM stalls).





#### 4. Interviewer Question: *"What if the user closes their browser tab or navigates away mid-stream?"*

* **Candidate Answer:**
* If the client disconnects, the server continues streaming tokens to a dead socket, wasting costly LLM inference tokens.
* **Solution:** FastAPI and async web servers monitor request cancellation (`await request.is_disconnected()`). Once detected, we cancel the downstream consumer task and send an immediate cancellation signal to the upstream LLM provider API to halt token generation.



---

## 3. Coding Problem 2: Batch Prompt Processing, Concurrency Semaphores & Exponential Jitter Backoff

### Base Problem Statement:

You have a list of 20 prompts that need to be processed concurrently via an LLM API. The system must enforce a **maximum concurrency limit of 5**, and handle `429 Too Many Requests` status errors using **exponential backoff with jitter**.

---

### Candidate Code Implementation & Production Engineering

```python
import asyncio
import random
import time
import logging
from typing import List, Dict, Any

logging.basicConfig(level=logging.INFO)

# Simulated LLM API call with rate-limiting and transient errors
async def mock_llm_api_call(prompt: str, sem: asyncio.Semaphore) -> Dict[str, Any]:
    async with sem:
        # Simulate rate-limiting failure (70% chance of 429 error on initial calls)
        if random.random() < 0.6:
            raise ConnectionError("HTTP 429: Too Many Requests (Rate limit exceeded)")
        
        await asyncio.sleep(0.5)  # Simulate processing delay
        return {"prompt": prompt, "response": f"Processed answer for: {prompt}"}


async def process_prompt_with_retry(
    prompt: str, 
    sem: asyncio.Semaphore, 
    max_retries: int = 4
) -> Dict[str, Any]:
    """
    Processes a single prompt with Semaphore concurrency control 
    and Exponential Backoff + Full Jitter.
    """
    delay = 1.0
    for attempt in range(1, max_retries + 1):
        try:
            return await mock_llm_api_call(prompt, sem)
        except (ConnectionError, TimeoutError) as err:
            if attempt == max_retries:
                logging.error(f"Prompt failed permanently after {max_retries} retries: {prompt}")
                return {"prompt": prompt, "error": str(err), "status": "failed"}
            
            # Full Jitter Backoff Formula: sleep = uniform(0, min(max_delay, base_delay * 2^attempt))
            calculated_delay = delay * (2 ** (attempt - 1))
            jittered_delay = random.uniform(0, calculated_delay)
            
            logging.warning(
                f"[Attempt {attempt}] Rate limited on '{prompt[:15]}...'. "
                f"Retrying in {jittered_delay:.2f}s..."
            )
            await asyncio.sleep(jittered_delay)


async def batch_process_prompts(prompts: List[str], max_concurrency: int = 5) -> List[Dict[str, Any]]:
    """
    Orchestrates batch execution of prompts using an asyncio Semaphore 
    and gathers results safely with return_exceptions=True.
    """
    sem = asyncio.Semaphore(max_concurrency)
    
    # Create tasks for all prompts
    tasks = [process_prompt_with_retry(prompt, sem) for prompt in prompts]
    
    # Execute concurrently
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    processed_results = []
    for idx, res in enumerate(results):
        if isinstance(res, Exception):
            processed_results.append({"prompt": prompts[idx], "error": str(res), "status": "exception"})
        else:
            processed_results.append(res)
            
    return processed_results

# Test Execution Sandbox
if __name__ == "__main__":
    test_prompts = [f"Prompt query number {i}" for i in range(1, 21)]
    
    start_time = time.time()
    loop = asyncio.get_event_loop()
    output = loop.run_until_complete(batch_process_prompts(test_prompts, max_concurrency=5))
    duration = time.time() - start_time
    
    logging.info(f"Batch completed in {duration:.2f} seconds with {len(output)} total items.")

```

---

### Follow-Up Questions & Production Edge Cases

#### 1. Interviewer Question: *"Why do we use Jitter in exponential backoff?"*

* **Candidate Answer:**
* Without jitter, if 50 concurrent worker threads receive a `429 Rate Limit` response simultaneously, they all calculate the exact same backoff delay (e.g., $2.0\text{s}$) and wake up simultaneously to retry. This creates a synchronized secondary spike known as the **Thundering Herd Problem**, immediately crashing the downstream API server again.
* Adding random jitter (`random.uniform(0, calculated_delay)`) spreads out retry attempts across a randomized time window, smoothing out traffic load.



#### 2. Interviewer Question: *"What errors should trigger a retry versus an immediate failure?"*

* **Candidate Answer:**
* **Retry:** Transient errors like `429 Too Many Requests`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`, and socket drop exceptions.
* **Do Not Retry:** Client-side configuration errors like `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, or `422 Unprocessable Entity` (e.g., malformed prompt structures or invalid API keys). Retrying these will produce identical failures.



#### 3. Interviewer Question: *"Is Concurrency Limit the exact same thing as Rate Limiting?"*

* **Candidate Answer:**
* **Crucial Distinction:**
* **Concurrency Limit** (controlled via our `asyncio.Semaphore(5)`) restricts how many tasks execute **at the exact same microsecond** (e.g., max 5 active HTTP calls running simultaneously).
* **Rate Limiting** restricts how many requests can be made **over a sliding time window** (e.g., 60 Requests Per Minute - RPM, or 40,000 Tokens Per Minute - TPM).


* Even with a concurrency limit of 5, if those 5 threads execute rapidly in succession, they can easily breach an RPM rate limit. Enterprise LLM gateways require a **Token Bucket** or **Leaky Bucket** rate limiter in addition to concurrency semaphores.
