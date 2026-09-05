---
title: "The Cheaper Agent That Wasn't: Mispricing Prompt-Cache Tokens"
collection: journal
order: 31
permalink: /journal/cache-token-mispricing/
excerpt: "A benchmark's headline finding — one agent dramatically cheaper than another — did not survive correct accounting. Cache-read tokens were priced at zero and cache-creation tokens at 1.0× instead of 1.25×, understating costs by up to 47%."
---

## Issue

Our web-search benchmark reported cost per question alongside accuracy — the whole point being that a compression result is meaningless without the bill attached.

It was **systematically under-counting cost** for any agent using prompt caching. The worst-affected agent was the one whose headline result we had already written up.

## Root Cause

Two omissions, both in the direction of looking cheaper.

**1. Cache-read tokens were tracked but never passed to the pricing function.** The field was populated on the usage record, carried all the way through the pipeline, and then silently dropped at the pricing boundary — the one place it mattered. Cache reads were **effectively free** in every report. They bill at 0.1× the input rate: cheap, not free, and at the hit rates a long-running agent achieves, the difference compounds into the dominant term.

**2. Cache-creation tokens weren't tracked at all.** One agent folded cache creation into ordinary `input_tokens`, pricing it at the **1.0× input rate instead of 1.25×** (5-minute TTL). Cache *writes* cost more than ordinary input — you are paying to populate the cache, not just to read the prompt — and this agent wrote a great many of them.

The bias is what makes this dangerous. Both errors understate cost, and they understate it **most for the agent that caches most aggressively** — which is exactly the agent that had won the comparison.

## Solution

Pricing gained the full model:

- The pricing function now takes cache-read tokens, cache-creation tokens **and a cache TTL**, applying the correct multipliers: **read 0.1×, write 1.25× at a 5-minute TTL, 2× at one hour**. The TTL parameter matters — the write premium is not a constant, and an agent that opts into longer-lived cache entries pays a different rate for the same tokens.
- The usage record gained a cache-creation field, and the cost breakdown forwards both cache fields rather than only the one it happened to know about.
- Every agent adapter now reads `cache_read_input_tokens` / `cache_creation_input_tokens` off the provider response, so no execution path is **dark on cache tokens**. This was the real gap: one adapter reported them, so the pipeline looked instrumented.

## The Result

Re-pricing the existing n=200 artifacts:

| Config | Reported | Corrected |
|---|---|---|
| Caching agent, compression ~1.4× | $0.135 | **$0.199** (+47%) |
| Caching agent, compression 2× | $0.115 | **$0.167** (+45%) |
| Caching agent, dynamic ratio | $0.088 | **$0.125** (+42%) |
| Non-caching agent, compression 2× | $0.340 | $0.340 (unaffected) |

End-to-end smoke on the same question, both agents answering correctly:

```
non-caching agent   $0.2418   (12 searches, 56s)
caching agent       $0.2954   (15 searches, 76s)
```

The ordering **reverses**. The agent that had looked dramatically cheaper is the more expensive one.

> **The headline finding does not survive correct accounting.**

That sentence went into the commit message and then into the results document. Retracting your own finding is the cheapest it will ever be at this stage; the alternative is a customer re-deriving it later.

## 💡 Takeaway

- **Cost is a measurement, and measurements have bugs.** We would never ship an accuracy number from an unvalidated scorer. Cost deserves the same scepticism.
- **Check the direction of an accounting error.** Both omissions understated cost, and understated it most for the agent that used caching hardest — the error was correlated with the treatment.
- **Re-price the artifacts, don't just fix the formula.** Storing raw token counts rather than computed costs made a full historical recompute possible without re-running a single question.
