# PwC Generative AI Engineer Walk-In Interview (Selected Candidate) — Questions & Answers Guide

---

## 1. Candidate Background & Interview Process Summary

* **Role & Company:** Generative AI Engineer at PricewaterhouseCoopers (PwC) — Selected Candidate Offer (Kolkata Walk-In).
* **Opportunity Source:** Direct recruiter reachout via Naukri for a Kolkata walk-in event.
* **Round Structure:**
* **Round 1 (Technical & System Design):** Self-introduction, agentic workflow drawing, end-to-end RAG chatbot system design, machine learning fundamentals, and a live Pandas coding challenge.
* **Round 2 (Technical Leadership & Fitment):** Discussion on project ownership, client training/mentorship capabilities, preferred work location, and compensation expectations.
* **Post-Interview:** HR document submission, CTC negotiation, and formal offer letter issuance within 15–20 days.



---

## 2. System Design & Agentic Architectures

### Question 1: Sketch and explain the end-to-end architecture of a production-grade RAG chatbot from user query input to final answer generation.

**Answer:**

```
                  [ User Query Input ]
                           │
                           ▼
               [ API Gateway & Auth ]
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
  [ Semantic Redis Cache ]     [ Query Pre-Processing ]
      (Cached Answer)          (Embedding & HyDE)
                                         │
                                         ▼
                           [ Scoped Vector Database ]
                             (ANN Similarity Search)
                                         │
                                         ▼
                           [ Cross-Encoder Reranker ]
                             (Top-K Document Chunks)
                                         │
                                         ▼
                           [ LLM Prompt Synthesis ]
                             (System Guardrails)
                                         │
                                         ▼
                           [ Groundedness Evaluator ]
                             (Fallbacks / SSE Stream)

```

1. **Ingestion & Indexing:** Raw documents are extracted, chunked (e.g., 500–1,000 tokens), embedded into dense vectors, and stored in a vector database alongside access-control metadata.
2. **Query Pipeline & Pre-Processing:** User query is authenticated via an API Gateway. Check a **Semantic Cache (Redis)** for existing answers. If missed, generate a query embedding (or apply Hypothetical Document Embeddings - HyDE).
3. **Retrieval & Reranking:** Perform Approximate Nearest Neighbor (ANN) vector search against the vector database using metadata filters. Run candidate chunks through a **Cross-Encoder Reranker** to select the top-$K$ most relevant passages.
4. **LLM Synthesis & Guardrails:** Pass context chunks and user query inside a grounded system prompt to the LLM. Run an automated **groundedness evaluation step** before streaming the response via Server-Sent Events (SSE).

---

### Question 2: How do you design and draw an Agentic AI workflow? How does an AI Agent differ from a standard linear pipeline?

**Answer:**

* **Standard Linear Pipeline:** Rigid execution sequence (`Input -> Search -> Prompt -> Output`) with zero dynamic adjustment or decision-making.
* **Agentic AI Workflow:** An autonomous loop driven by an LLM acting as a reasoning engine:
* **Planning / ReAct Loop:** Model breaks down a complex query into explicit sub-tasks (`Thought -> Action -> Observation`).
* **Tool Routing:** Dynamically calls tools (e.g., SQL execution, Web Search, Calculator API) using structured JSON schemas based on user context.
* **Reflection & Fallback:** Evaluates tool execution outputs. If output is invalid or incomplete, it self-corrects and iterates up to a maximum step limit before delivering the final response.



---

## 3. Traditional Machine Learning Fundamentals

### Question 3: Why are traditional ML concepts (Classification, Regression, Cost Functions, Loss Functions) still asked in Generative AI interviews?

**Answer:**

* Gen AI solutions do not exist in isolation. Pre-retrieval query routing, content moderation, document categorization, and vector reranking often rely on classical supervised ML models (classifiers) rather than heavy LLM calls to keep latency and costs low.
* **Key Definitions:**
* **Loss Function vs. Cost Function:** Loss function measures prediction error for a *single* data instance (e.g., Binary Cross-Entropy, MSE). Cost function is the average loss across the *entire dataset*.
* **Classification vs. Regression:** Classification predicts discrete class labels (e.g., routing a user query to "Legal" vs "HR"); Regression predicts continuous numerical outputs.



---

### Question 4: How do you detect and handle outliers in a dataset before feeding data into ML/Embedding models?

**Answer:**

1. **Detection Techniques:**
* **Z-Score Method:** Identifies data points lying beyond 3 standard deviations ($\lvert Z \rvert > 3$) from the mean for normally distributed data.
* **IQR (Interquartile Range):** Defines outliers as points falling below $Q_1 - 1.5 \times \text{IQR}$ or above $Q_3 + 1.5 \times \text{IQR}$ (robust for non-normal distributions).
* **Isolation Forests:** Unsupervised tree algorithm ideal for high-dimensional anomaly detection.


2. **Treatment Methods:**
* **Trimming/Removal:** Dropping verified corrupt or erroneous records.
* **Winsorization (Capping):** Capping extreme values at set percentiles (e.g., 1st and 99th percentiles) to retain data volume without skewing feature scales.



---

## 4. Python & Pandas Live Coding Challenge

### Question 5: Coding Task: Given a Pandas DataFrame with columns `'Student'` and `'Marks'`, filter out all students with marks less than 30 and update/add a new column `'Result'` with the value `'Fail'`.

```python
import pandas as pd

# Sample Data Setup
data = {
    'Student': ['Alice', 'Bob', 'Charlie', 'David', 'Eve'],
    'Marks': [85, 22, 45, 12, 67]
}
df = pd.DataFrame(data)

# Vectorized pandas operation to assign 'Pass'/'Fail' based on threshold < 30
df['Result'] = 'Pass'
df.loc[df['Marks'] < 30, 'Result'] = 'Fail'

print(df)

```

**Output:**

```text
   Student  Marks Result
0    Alice     85   Pass
1      Bob     22   Fail
2  Charlie     45   Pass
3    David     12   Fail
4      Eve     67   Pass

```

---

## 5. Rapid Preparation Checklist for Urgent Gen AI Interviews

If you have an interview scheduled within **24–48 hours**, prioritize this high-impact checklist:

1. **End-to-End RAG Architecture:** Practice sketching a complete architecture diagram on paper (Ingestion $\rightarrow$ Embeddings $\rightarrow$ Vector DB $\rightarrow$ Reranker $\rightarrow$ LLM $\rightarrow$ Guardrails).
2. **Agentic Framework Concepts:** Be prepared to explain multi-agent orchestration, tool calling, memory management (short-term vs. long-term), and planning loops (ReAct).
3. **ROI & Business Value:** Formulate clear explanations of how your architecture optimizes cost (token usage, caching) and latency (streaming, quantization, TTFT).
4. **Python Data Manipulation:** Refresh vectorized Pandas operations (`loc`, `iloc`, filtering, `groupby`), generators, decorators, and async processing routines.
