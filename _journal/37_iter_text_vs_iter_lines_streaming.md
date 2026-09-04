---
title: "Streaming Yielded Zero Chunks: iter_text() Is Not iter_lines()"
collection: journal
permalink: /journal/iter-text-is-not-iter-lines/
excerpt: "A server-sent-events parser iterated decoded text chunks instead of lines, so a data frame split across two chunks was never recognised. Tests passed because a whole frame happened to land in one chunk every time — production traffic is not so accommodating."
---

## Issue

A streaming call returned HTTP 200 and yielded nothing. No exception, no partial output — the stream simply produced zero events.

## Root Cause

The parser iterated the response using a method that yields arbitrarily sized, arbitrarily bounded chunks of decoded text — not lines. Server-sent events are line-oriented: each event is a `data: {...}` frame terminated by a newline, and nothing about a text-chunk iterator guarantees a frame arrives whole. A single frame split across two chunks meant the fragment in each chunk never matched the pattern the parser was checking for, and both fragments were silently discarded.

The bug passed every existing test, because in a test environment a small response body typically arrives as one chunk — the whole frame lands together by accident, every time, until it doesn't. Nothing distinguished "this works" from "this happens to work on small inputs."

A missing header compounded it: without explicitly requesting the event-stream content type, some intermediaries were free to alter chunking behavior in ways a strict line-based parser would tolerate and a chunk-based one would not.

A second act followed once the fix shipped. Switching to a proper line iterator fixed the boundary problem, but that convenience method buffers an unterminated line with no upper bound — a broken or hostile server sending one endless line without a newline could exhaust the client's memory. The line iterator was replaced again with a hand-written version that reads raw bytes, splits on newline itself, and raises once a single line exceeds a fixed cap.

## Solution

Parse by line, not by arbitrary chunk boundary, and request the event-stream content type explicitly. Bound the line length so a malformed or adversarial stream cannot turn a client into an unbounded buffer. On any frame that fails to parse as expected JSON, surface the raw chunk rather than silently skipping it — silence is exactly what let the original bug hide.

## 💡 Takeaway

- **"Iterate text" and "iterate lines" are not interchangeable on a chunked transport.** A protocol defined in terms of lines needs a line-oriented reader; anything else is correct only by coincidence of how the data happened to be chunked.
- **A test that passes because a whole message happened to fit in one read is not testing the protocol — it's testing the test's own data size.** Streaming logic needs a test that forces a message across an artificial chunk boundary.
- **Fixing an obvious bug can introduce a subtler one at the next layer of convenience.** A method that solves "read by line" by buffering indefinitely trades a correctness bug for an availability bug; check what your new primitive does on the pathological input before trusting it.
