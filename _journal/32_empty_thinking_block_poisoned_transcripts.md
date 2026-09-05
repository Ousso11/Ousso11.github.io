---
title: "Empty Thinking Blocks Permanently Poisoned Conversation Transcripts"
collection: journal
permalink: /journal/fixing-the-writer-doesnt-unpoison-old-data/
excerpt: "An SSE resynthesis path emitted a whole thinking block without any of the delta events a spec-compliant client uses to build it. Clients reconstructed an empty block with a valid signature, saved it to disk, and every future replay of that conversation was rejected forever."
---

**The trick:** fixing a buggy writer doesn't repair data it already wrote. Anything with durable derived state needs a read-side repair pass too — proven, by test, to be a byte-identical no-op on input that was already clean.

## Issue

An agent's benchmark pass rate dropped by nearly a quarter, and the affected sessions did not recover. Every retry, every restart, every future request against that exact conversation returned the same rejection: *"each thinking block must contain thinking."* Permanently.

## Root Cause

One code path re-synthesizes a streaming response from a buffered JSON one — necessary whenever an intercepted request has to be resent non-streaming and then replayed to the client as if it had streamed. The synthesizer handled ordinary content correctly. Reasoning ("thinking") blocks fell into a catch-all branch that emitted the entire block inside a single start event, with **none of the incremental delta events** that a compliant client actually uses to build the block's content.

A spec-compliant client does not read the start event's payload for content — it accumulates content strictly from the deltas. With no deltas, the client reconstructed a thinking block with **empty content and a valid-looking signature**, and persisted exactly that to its on-disk transcript.

The transcript was now permanently damaged. Every future turn replays the full conversation history back upstream, including that empty-but-signed block, and the upstream API correctly rejects a thinking block with no thinking in it. There was no way to recover — the bad data was already saved.

## Solution

Two fixes, because fixing the generator does not repair what it already wrote.

**Fix the writer.** The synthesizer now emits a proper sequence: a start event with an empty shell, a delta carrying the actual thinking text, a delta carrying the signature, then a stop event — mirroring exactly what a live stream would have produced. A separate, opaque variant of thinking content passes through with no deltas at all, matching how that variant is defined to work.

**Fix the reader.** A sanitizer runs on every inbound request, before anything else touches it: it strips empty thinking blocks out of assistant messages, and if a message's only content was such a block, it drops the whole message rather than leaving an empty content array (which is itself invalid — dropping is legal, an empty array is not). This repairs transcripts that were already poisoned, on every future turn, without anyone having to identify and fix them individually.

The sanitizer has one hard requirement: on a transcript that needs no repair, it must return **byte-identical** output. Anything else would invalidate the provider's prompt cache on every single request, trading a correctness bug for a cost regression.

## 💡 Takeaway

- **When your output becomes someone else's durable input, fixing the generator only stops new damage.** Existing damage needs a repair path on the input side, because there is no way to reach back into storage you don't control.
- **A resynthesis or transcoding layer must reproduce the protocol's actual construction rules, not just its final shape.** "The parsed content matches" and "the wire format is legal" are different claims, and clients that build state incrementally only care about the second one.
- **Any repair pass injected into a hot path must be a proven no-op on clean input.** State that byte-identical output as an explicit, tested invariant — it's the only thing standing between a correctness fix and a cache-invalidation regression.
- **A permanent, non-recoverable failure mode is worth hunting for specifically.** Most bugs degrade a request; this one contaminated a conversation for its entire remaining life. That asymmetry is what made a partial defect look like a catastrophic regression.
