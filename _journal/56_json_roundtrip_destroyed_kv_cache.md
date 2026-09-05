---
title: "Nothing Changed in Meaning. The Cache Missed Every Time."
collection: journal
permalink: /journal/parse-mutate-reserialize-breaks-prefix-caches/
excerpt: "Prompt-cache hit rates collapsed on any request that passed through one code path, with no error and no log line. Parsing the request into a generic map, appending a message, and re-serializing produced byte-for-byte different output from byte-for-byte identical meaning — and a prefix cache only cares about bytes."
---

**The trick:** anything keyed on a byte prefix — an LLM provider's prompt cache, a CDN, content-addressed storage — cannot survive a parse-mutate-reserialize round trip through a generic data structure. Map keys get reordered, field order isn't preserved, whitespace changes. Edit the bytes you need to change and leave everything else untouched.

## Issue

Requests that passed through a specific request-modification step lost their prompt-cache hit rate almost entirely. There was no error, no warning, and nothing in either the request or the response looked wrong — the modified request was semantically identical to what it should have been. The only symptom was a cost and latency regression invisible to anything except a provider-side cache-hit metric nobody was watching closely enough, for long enough, to catch immediately.

## Root Cause

The step in question needed to append one message to an existing request body. The implementation did the obvious thing: parse the JSON into a generic key-value structure, append the new message to the messages array, re-serialize back to JSON.

That round trip is semantically lossless and byte-for-byte destructive. A generic JSON serializer commonly emits object keys in a canonical order — frequently alphabetical — rather than preserving whatever order the original bytes were in, and parsing into a generic map discards the original ordering entirely; there's nothing left to preserve it from. The re-serialized request meant exactly the same thing and was a completely different sequence of bytes from the very first character.

A prompt cache keyed on byte-identical prefixes doesn't know or care about semantic equivalence. It compares bytes. A request that used to share a long common prefix with every other request in the conversation now diverged at byte zero, and the *entire* prompt — not just the appended message — missed the cache on every single request that passed through this step.

## Solution

Stop parsing into a generic structure at all. Edit the JSON bytes surgically: locate the exact byte range that needs to change — appending one element to an array — and modify only that range, leaving every other byte, including field order and whitespace, untouched. Purpose-built libraries for exactly this exist in most languages; reach for one instead of a generic parse/mutate/serialize round trip whenever the untouched bytes need to stay untouched.

The regression test that makes this durable doesn't check the JSON for semantic equality — it checks that everything before the intended change is **byte-identical** to the original. That's the property a prefix cache actually depends on, and it's a different, stricter claim than "still valid JSON meaning the same thing."

## 💡 Takeaway

- **Semantic equivalence and byte equivalence are different properties, and a prefix-keyed cache only cares about the second one.** Any code that touches a request destined for a cached provider needs to preserve bytes it isn't intentionally changing, not just preserve meaning.
- **Parsing into a generic, order-agnostic structure (a map, a dict) is lossy for anything downstream that depends on order.** The data survives; the byte layout doesn't. If the byte layout matters, don't go through that structure.
- **This failure mode produces no error and no diagnostic — only a cost and latency regression that looks like normal variance until someone specifically checks the cache-hit rate.** Any system relying on prefix caching deserves a monitored metric for hit rate, not just for cost, because a silent full-prefix miss is invisible everywhere else.
- **Test for byte-identical output on the unmodified portion, not just "still parses to the same thing."** A round-trip test that re-parses both sides and compares the parsed structures will pass on exactly this bug.
