---
title: "A Compression Ratio of Exactly 1.000×"
collection: journal
permalink: /journal/json-shaped-input-no-op/
excerpt: "Every compress() call in the agent middleware returned its input unchanged — ratio 1.000× — while genuinely reaching the server and returning 200. The compressor no-ops on JSON-shaped input, and the search tool was handing it raw JSON."
---

## Issue

The agent middleware was compressing web-search results before they entered the model's context. The requests were going out, the server was returning 200, and the measured ratio was **1.000×**. Every time.

An exact 1.000 is not a weak result. A weak result looks like 1.03×. An exact 1.000 means the pipeline is structurally doing nothing, and the only question is where.

## Root Cause

`WebSearchTool.tavily` returned the raw dictionary from `langchain_tavily.TavilySearch`:

```json
{"query": "...", "results": [{"title": ..., "url": ..., "content": ...}]}
```

The compression backend **silently no-ops on JSON-shaped input.** It is a token-level compressor over prose; handed a serialised structure, dropping tokens would produce invalid JSON, so it returns the input unchanged rather than corrupting it. Reasonable behaviour — expressed as a success response with no signal that nothing happened.

A direct probe with the SDK isolated it cleanly, and this is the step that turned a guess into a fact:

| Same content, sent as | Ratio |
|---|---|
| JSON | **1.000×** |
| Plain text | **1.94×** |

Identical bytes of information. The only variable was the shape.

The neighbouring Brave search tool already had this fix. Tavily had been added later without it — the kind of divergence that appears whenever the same idea is implemented twice.

## Solution

Wrap `TavilySearch` in a `StructuredTool` that:

- calls the underlying `.invoke({query})`;
- **flattens** `{results: [{title, url, content}, ...]}` into blank-line-separated `title\nurl\ncontent` blocks — the Brave-style shape the compressor actually understands;
- declares an explicit **`query` schema**, so the model emits well-formed tool calls (parity with Brave, which already had it);
- falls back to `JSON.stringify` for unknown response shapes, so a future Tavily change degrades to today's behaviour instead of throwing.

Live re-test against the production API: **386 → 242 tokens, 1.60×**, against 1.000× before.

Tests were added in both SDKs for plain-text flattening, the unknown-shape fallback, and query-schema parity. Two existing kwarg-threading tests were rewritten against a stub `TavilySearch`, so they assert that kwargs reach the underlying constructor instead of poking at the wrapper — the tests had been coupled to the shape of the thing being replaced.

## 💡 Takeaway

- **A no-op that returns 200 is indistinguishable from success without a ratio.** Measuring the effect, not just the call, is what surfaced this at all.
- **Compare against the same content in a different shape.** Holding information constant and varying only encoding turned an ambiguous number into a one-line diagnosis.
- **Two implementations of one idea will diverge.** Brave had the fix; Tavily didn't. Parity between sibling integrations deserves an explicit test.
