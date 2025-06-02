---
title: "Scaling Data Mining with API Efficiency Under TPM Limits"
collection: journal
excerpt: "Efficiently mining structured text or graphs using GPT-4 APIs while staying under 2M TPM."
---

## Issue

We needed to **mine and transform** large text datasets into structured formats like:

- **Structured JSON from raw text**
- **Graphs from code or document relationships**

However, **GPT-4's 2M TPM constraint** and high latency posed a scaling bottleneck.

title: "Scaling Data Mining with API Efficiency Under TPM Limits"
collection: journal
excerpt: "Efficiently mining structured text or graphs using GPT-4 APIs while staying under 2M TPM."

## Issue

We needed to **mine and transform** large text datasets into structured formats like:

- **Structured JSON from raw text**
- **Graphs from code or document relationships**

However, **GPT-4's 2M TPM constraint** and high latency posed a scaling bottleneck.

## Solution

We implemented a **parallel, optimized API pipeline** using LangChain, with full token tracking and queue management:

### 1. Query Batching with Subtasks

- Split tasks into **independent subtasks** (e.g., paragraph → entity list)
- Batched multiple prompts per request using `LangChain.map_reduce`
- Used OpenAI **function-calling** API to enforce structured output
- Compressed prompts before each call to fit more per request

### 2. TPM-Conscious Scheduling

- Used **LangChain’s token-aware throttling** to avoid rate limit breaches
- Distributed load across **multiple API keys/orgs** to parallelize requests
- **Tracked token usage per key** using a `token_window` sliding structure
- Maintained a **pending job queue** with TTL and retries

### 3. Throttling Logic with Queue and Token Tracker

- Used a `deque`-based job queue for pending transformations
- Created a **`token_usage_log = {api_key: [(timestamp, token_count), ...]}`** structure
- In each loop:

```python
while job_queue:
    job = job_queue.popleft()
    if not token_within_limit(api_key):
        job_queue.append(job)  # Requeue if over TPM
        sleep(1)
        continue

    output = call_openai(job)
    log_token_use(api_key, token_count)
    save_output(output)
```

