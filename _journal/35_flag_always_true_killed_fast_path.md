---
title: "A Miswired Feature Flag Disabled Streaming's Fast Path for Months"
collection: journal
order: 17
permalink: /journal/a-flag-that-never-varies-is-a-lie/
excerpt: "Time-to-first-token equalled full generation time for every streaming request, for months. A fast pass-through path existed and was correctly written — gated on a flag that had been set unconditionally on every request, so the branch was unreachable code nobody noticed."
---

**The trick:** trace a flag to everywhere it's set, not just where it's read. A boolean that never varies is worse than no boolean, because it reads as a decision that isn't actually being made.

## Issue

Every streaming request behaved as if fully buffered — time-to-first-token equalled total generation time, for months — despite a correctly written direct pass-through path sitting right there in `handleStreamingWithExpand`.

## Root Cause

The fast path was gated on `!pipeCtx.PhantomToolsInjected`. A related feature set that flag to `true` on *every* request, whether tool discovery was enabled or not — so the condition never varied, and the fast branch was dead code that looked correct on review.

## Solution

Set the flag from the real injection count instead of unconditionally, and split the handler into two genuine implementations: a direct per-chunk `Write`+`Flush` path, and a buffered path used only when `ToolsFiltered || PhantomToolsInjected`. Measured time-to-first-token after the fix: **0.14s**, down from matching full generation time.

## 💡 Takeaway

- A boolean that never varies is worse than none — it reads as a decision being made when it isn't.
- Test that a performance-gated branch is actually taken under realistic conditions, not just that both branches are individually correct.
- Trace a flag to every place it's set before trusting what its read site implies.
