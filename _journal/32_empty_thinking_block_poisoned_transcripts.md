---
title: "[6] Malformed SSE Reconstruction Permanently Corrupted Conversation History"
collection: journal
order: 6
permalink: /journal/fixing-the-writer-doesnt-unpoison-old-data/
excerpt: "An SSE resynthesis path emitted a whole thinking block without any of the delta events a spec-compliant client uses to build it. Clients reconstructed an empty block with a valid signature, saved it to disk, and every future replay of that conversation was rejected forever."
---

**The trick:** fixing a buggy writer doesn't repair data it already wrote. Anything with durable derived state needs a read-side repair pass too — proven to be a byte-identical no-op on already-clean input.

## Issue

An agent's benchmark pass rate dropped sharply, and the affected sessions never recovered. Every retry against those exact conversations failed the same way, permanently.

## Root Cause

A streaming-resynthesis path emitted an entire reasoning block in one event instead of the incremental deltas a compliant client uses to build it. Clients reconstructed an empty block with a valid signature and saved it to disk — and every future turn replayed that poisoned block upstream, which correctly rejected it every time.

## Solution

Fix the generator to emit proper deltas. Separately, add a sanitizer on every inbound request that strips empty thinking blocks from history — repairing already-poisoned transcripts without anyone finding them individually. The sanitizer must be byte-identical on clean input, or it breaks prompt caching.

## 💡 Takeaway

- Fixing the writer only stops new damage — durable data already written needs its own repair path.
- A protocol resynthesizer must reproduce the actual construction rules, not just the final shape.
- Any repair pass on a hot path must be a proven no-op on clean input.
