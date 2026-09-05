---
title: "The Fast Path That Never Once Ran, for Months"
collection: journal
permalink: /journal/a-flag-that-never-varies-is-a-lie/
excerpt: "Time-to-first-token equalled full generation time for every streaming request, for months. A fast pass-through path existed and was correctly written — gated on a flag that had been set unconditionally on every request, so the branch was unreachable code nobody noticed."
---

**The trick:** trace a flag to everywhere it's set, not just where it's read. A boolean that never varies is worse than no boolean, because it reads as a decision that isn't actually being made.

## Issue

Every streaming request behaved as if it were fully buffered: the client received nothing until the entire response had been generated, and time-to-first-token equalled total generation time. There was, in the same code, a direct pass-through path clearly designed to stream chunks to the client as they arrived. It was never running.

## Root Cause

The streaming handler had a real branch: if nothing about the request required rewriting the response, forward each chunk immediately as it arrived. The condition guarding that branch checked a flag meaning, in effect, "did we inject anything into this response that needs buffering to process."

The flag was set to true unconditionally, on every request — because a related feature injected some scaffolding whether or not the thing it supported was actually enabled for that request. The flag was answering a different question than the one it was named for, and its actual value never varied. A branch guarded by a condition that is always true and a branch guarded by no condition at all produce identical behavior; the difference is only that the first one reads, incorrectly, as if a decision were being made.

Nobody caught it because the buffered path is also correct — just slow. There was no crash, no error, no wrong output. Only a fast path that had quietly been dead code for as long as the flag had existed.

## Solution

Two changes, at two different levels. First, make the flag honest: only set it when the feature it names is genuinely active for this request, driven by the real count of things injected rather than a blanket assumption. Second, split the handler cleanly into two real implementations — a direct path that writes and flushes each chunk as it streams through, computing any bookkeeping by observing the pass-through rather than buffering it, and a buffered path used only when the response genuinely needs post-processing before it can be sent.

The comment left behind explains why buffering is unavoidable in that second case: once bytes have been forwarded to the client, they cannot be recalled, so any decision requiring the whole response has to be made before anything is sent.

## 💡 Takeaway

- **A boolean that never varies is worse than no boolean at all.** It makes a reader believe a decision is happening when the branch has silently degenerated into dead code — and a reviewer checking "is there a fast path" will find one, read it, and conclude it works.
- **Whenever a flag gates a performance-sensitive branch, test that the branch is actually taken under realistic conditions**, not just that both branches produce correct output in isolation. Correctness tests will not catch a fast path that never fires.
- **Trace a flag back to everywhere it is set, not just to where it is read.** The bug was invisible from the read site; it only became obvious once someone asked "under what conditions is this false."
