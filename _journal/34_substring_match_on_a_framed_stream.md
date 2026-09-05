---
title: "The Detector That Fired on Talk and Missed the Real Thing"
collection: journal
order: 18
permalink: /journal/substring-match-on-a-framed-protocol/
excerpt: "Detecting a specific kind of tool call in a streaming response with a raw substring search caused duplicate upstream billing whenever the model merely mentioned the tool's name in prose, and missed real calls whose name was split across a chunk boundary."
---

**The trick:** never use substring matching to detect structure in a framed protocol. Parse the frames properly, and keep any substring check as a cheap pre-filter only — never as the decision itself.

## Issue

A detector watching streamed SSE responses for a specific tool call — `bytes.Contains(chunk, []byte(toolName))` — was both too eager and not eager enough. Some requests triggered a duplicate upstream call for no reason; real calls slipped through undetected.

## Root Cause

The model *saying* "I'll use Read next" in a text delta matched the same substring check as an actual `content_block_start` with `type: "tool_use"`. And a tool name split across two `resp.Body.Read` chunk boundaries matched neither chunk at all — the detector never saw it whole.

## Solution

Parse the buffered SSE events structurally: match on `content_block.type == "tool_use"` (Anthropic) or `choices[].delta.tool_calls[]` (OpenAI-shaped), never on raw bytes. The substring check survives only as a cheap pre-filter before the real parse.

## 💡 Takeaway

- Substring matching on a framed protocol breaks the moment a pattern spans a chunk boundary.
- Raw text output from a language model will eventually match prose that means nothing structurally.
- Keep the cheap check as a pre-filter only, never as the actual decision.
