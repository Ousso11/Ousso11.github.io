---
title: "The Server That Restarting Couldn't Fix"
collection: journal
permalink: /journal/derived-state-not-latched-flags/
excerpt: "An on-premise instance whose telemetry uploads kept failing began returning 503 and never stopped. Restarting did not help; shipping a new image did not help. The health anchor was a latched flag, and the only code that cleared it could no longer run."
---

**The trick:** replace a latched health/gate flag with a value derived live from the data it describes — a derived value can't get stuck set or stuck clear.

## Issue

An on-premise instance stopped serving entirely. Restarting didn't help. A fresh image didn't help either. It had bricked itself with no recovery path short of hand-editing state on disk.

## Root Cause

The health gate was a flag, set when the event queue fell too far behind and cleared only by the code path that drains it successfully. But failed events past a retry cap were *deleted*, not delivered — so a queue that emptied by deletion never ran the clearing path, and the flag stayed set forever, reloaded from disk on every restart. Two more bugs compounded it: every failure was treated as retryable, so one permanently-rejected batch blocked the whole queue; and failures had no jitter, so the whole fleet retried in lockstep.

## Solution

Derive the health gate from the queue itself — the age of the oldest unreported item — instead of a flag nothing can reliably clear. Classify failures into retryable vs. terminal, quarantine what the server keeps rejecting, and add jitter to retries.

## 💡 Takeaway

- Prefer derived state to latched flags for anything gating service — a flag has failure modes a live query doesn't.
- "The queue is empty" and "the work was done" are different facts if your cleanup path can discard.
- A fail-closed gate must never be able to reject the exact request that would clear it.
