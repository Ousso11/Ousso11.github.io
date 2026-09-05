---
title: "The Detector That Fired on Talk and Missed the Real Thing"
collection: journal
permalink: /journal/substring-match-on-a-framed-protocol/
excerpt: "Detecting a specific kind of tool call in a streaming response with a raw substring search caused duplicate upstream billing whenever the model merely mentioned the tool's name in prose, and missed real calls whose name was split across a chunk boundary."
---

**The trick:** never use substring matching to detect structure in a framed protocol. Parse the frames properly, and keep any substring check as a cheap pre-filter only — never as the decision itself.

## Issue

A detector watching streamed responses for a specific tool call was both too eager and not eager enough. Some requests triggered a duplicate upstream call for no reason; others let the exact pattern it was built for slip through.

## Root Cause

The detector was a raw substring search over each chunk's bytes. The model *talking about* a tool matched just as well as an actual call, and a tool name split across a chunk boundary matched neither chunk at all.

## Solution

Parse the buffered stream events structurally and detect the real protocol shape for each provider format. The substring check survives only as a cheap pre-filter before the real parse — never the decision.

## 💡 Takeaway

- Substring matching on a framed protocol breaks the moment a pattern spans a chunk boundary.
- Raw text output from a language model will eventually match prose that means nothing structurally.
- Keep the cheap check as a pre-filter only, never as the actual decision.
