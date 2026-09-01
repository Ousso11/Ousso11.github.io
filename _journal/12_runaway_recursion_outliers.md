---
title: "5% of Rows Cost $10–60 Each: An Unset Recursion Limit"
collection: journal
permalink: /journal/langgraph-recursion-outliers/
excerpt: "A benchmark's cost-per-question quadrupled on one config. The agent never wired its max_steps into LangGraph, so it ran at the default recursion_limit=25 — and on hard questions kept looping with growing context, producing rows with 5–20M input tokens."
---

## Issue

A benchmark sweep showed one configuration at **$1.77 per question** against a typical **$0.43** — a 4× cost blowout on a run that was supposed to be measuring the *savings* from compression.

The mean was lying. This was not a config that was uniformly four times more expensive; it was a config where about **5% of rows cost $10–60 each** and the rest were normal. Two such outliers in the first 30 questions were enough to quadruple the average.

Heavy-tailed cost distributions break every summary statistic you would normally reach for, and they look like a systematic regression right up until you plot them.

## Root Cause

The agent **never wired its own `max_steps` config into LangGraph.** The config field existed, was documented, was set in every experiment YAML — and was read by nothing.

So the agent ran at LangGraph's default **`recursion_limit=25`**.

On hard BrowseComp questions the failure mode compounds:

1. the agent exhausts its `max_searches` budget (12) without finding an answer;
2. it keeps iterating anyway, up to the framework's own limit;
3. each iteration carries **the entire accumulated context forward**;
4. input tokens grow superlinearly — the worst rows reached **5–20M input tokens**.

An unbounded loop over an append-only context is not a linear cost overrun. It is quadratic, and it only triggers on the questions that are hard enough to exhaust the budget — which is why it was invisible in smoke tests and appeared only in full sweeps on the hardest dataset.

## Solution

Wire the config that was already there:

```python
client.messages.create(..., config={"recursion_limit": max_steps})
```

The SDK already forwards a `config` kwarg straight through to `agent.invoke`, so no plumbing was needed — only the connection nobody had made.

**Mapped 1:1, not 2×.** The configured `max_steps` is 20, and using it directly keeps the cap tight enough to bound the worst outliers while still leaving room for 12 searches plus a final answer. A 2× safety margin here would have preserved most of the tail it was meant to remove.

Smoke test on the affected config (n=2): **$0.43/question**, back in the normal band, against $1.77 on the n=30 partial.

## 💡 Takeaway

- **A config field that nothing reads is worse than a missing one.** Everyone assumed the cap was in effect because it was written down.
- **Look at the distribution before believing the mean.** "This config is 4× more expensive" and "this config has a 5% tail of catastrophic rows" call for completely different fixes.
- **Agent loops need a hard framework-level bound, not just a semantic one.** `max_searches` limited what the agent was *supposed* to do. Only `recursion_limit` limited what it *could* do.
- **The default is a decision you made by not making it.** `recursion_limit=25` was chosen by LangGraph for a different context and inherited silently.
