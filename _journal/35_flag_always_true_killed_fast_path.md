---
title: "The Fast Path That Never Once Ran, for Months"
collection: journal
permalink: /journal/a-flag-that-never-varies-is-a-lie/
excerpt: "Time-to-first-token equalled full generation time for every streaming request, for months. A fast pass-through path existed and was correctly written — gated on a flag that had been set unconditionally on every request, so the branch was unreachable code nobody noticed."
---

**The trick:** trace a flag to everywhere it's set, not just where it's read. A boolean that never varies is worse than no boolean, because it reads as a decision that isn't actually being made.

## Issue

Every streaming request behaved as if fully buffered — time-to-first-token equalled total generation time, for months, despite a correctly written fast path sitting right there in the code.

## Root Cause

The fast path was gated on a flag meaning "did we inject anything requiring buffering." A related feature set that flag to true on every single request, whether or not it actually applied — so the condition never varied, and the fast branch was dead code that looked, on review, exactly like a working one.

## Solution

Make the flag honest — set only when the feature is genuinely active, driven by a real count rather than a blanket assumption — and split the handler into two real implementations: direct pass-through, and buffered only when genuinely needed.

## 💡 Takeaway

- A boolean that never varies is worse than none — it reads as a decision being made when it isn't.
- Test that a performance-gated branch is actually taken under realistic conditions, not just that both branches are individually correct.
- Trace a flag to every place it's set before trusting what its read site implies.
