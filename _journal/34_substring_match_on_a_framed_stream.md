---
title: "bytes.Contains on a Streaming Protocol Finds Prose, Misses Tool Calls"
collection: journal
permalink: /journal/substring-match-on-a-framed-protocol/
excerpt: "Detecting a specific kind of tool call in a streaming response with a raw substring search caused duplicate upstream billing whenever the model merely mentioned the tool's name in prose, and missed real calls whose name was split across a chunk boundary."
---

## Issue

A detection routine watching a streaming response for a specific pattern of tool call was both too eager and not eager enough. Some requests triggered a full duplicate upstream call for no visible reason. Separately, the thing the detector existed to catch sometimes slipped through undetected.

## Root Cause

The detector was a single line: search the raw bytes of each streamed chunk for the tool's name as a substring.

That has two independent failure modes on a framed, chunked protocol.

**False positives.** The model does not only *call* tools — it frequently talks about them. A text delta saying "I'll use the search tool next" contains the tool's name as a substring with zero relationship to an actual tool call. Every such mention triggered the full duplicate-request logic meant only for genuine calls.

**False negatives.** A streaming response arrives as a sequence of arbitrarily sized chunks, and nothing guarantees a tool's name lands entirely within one chunk. A name split across a chunk boundary is invisible to a substring search running independently on each chunk — the very case the detector was built for could pass through unnoticed.

## Solution

Replace substring matching with structural matching. The buffered stream events are parsed properly, and detection asks the actual protocol question for each format in use: is this a tool-call content block starting, is this a tool-call delta in the chat-completions shape, is this a function-call item being added or completed in the newer responses shape. Text and reasoning deltas are never inspected at all, so no amount of a model talking about a tool can trigger anything.

The substring check survives only as a cheap pre-filter to skip obviously irrelevant chunks before doing the real parse — never as the decision itself.

## 💡 Takeaway

- **Substring matching on a framed protocol is a bug waiting for a chunk boundary.** Any pattern that can legitimately span two reads will eventually be split by one, and a per-chunk check has no way to notice.
- **A raw text match on a language model's output will always eventually match prose.** If the same tokens can appear as *content* and as *structure*, only a structural parse can distinguish the two.
- **It's fine to keep the cheap check — just relegate it to filtering, not deciding.** A fast reject that occasionally lets a false positive through to the slow, correct path is harmless. A fast decision that's occasionally wrong is not.
