---
title: "Nothing Changed in Meaning. The Cache Missed Every Time."
collection: journal
permalink: /journal/parse-mutate-reserialize-breaks-prefix-caches/
excerpt: "Prompt-cache hit rates collapsed on any request that passed through one code path, with no error and no log line. Parsing the request into a generic map, appending a message, and re-serializing produced byte-for-byte different output from byte-for-byte identical meaning — and a prefix cache only cares about bytes."
---

**The trick:** anything keyed on a byte prefix — an LLM provider's prompt cache, a CDN, content-addressed storage — cannot survive a parse-mutate-reserialize round trip through a generic data structure.

## Issue

Prompt-cache hit rates collapsed on any request passing through one specific step, with no error anywhere — just a silent cost and latency regression.

## Root Cause

The step parsed a request into a generic JSON map, appended a message, and re-serialized. Generic serializers commonly reorder object keys, and parsing into a map discards the original order entirely — the output meant exactly the same thing and was a different sequence of bytes from character one. A prefix cache compares bytes, not meaning, so the *entire* prompt missed on every request through that path.

## Solution

Edit the JSON bytes surgically instead of parsing into a generic structure — change only the exact range that needs to change, leaving everything else byte-identical.

## 💡 Takeaway

- Semantic equivalence and byte equivalence are different properties; a prefix-keyed cache only cares about the second.
- Parsing into a generic map is lossy for anything downstream that depends on byte order.
- Test for byte-identical output on the untouched portion, not just "still parses to the same thing."
