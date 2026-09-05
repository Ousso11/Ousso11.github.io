---
title: "The Server That Restarting Couldn't Fix"
collection: journal
order: 9
permalink: /journal/derived-state-not-latched-flags/
excerpt: "An on-premise instance whose telemetry uploads kept failing began returning 503 and never stopped. Restarting did not help; shipping a new image did not help. The health anchor was a latched flag, and the only code that cleared it could no longer run."
---

**The trick:** replace a latched health/gate flag with a value derived live from the data it describes — a derived value can't get stuck set or stuck clear.

## Issue

An on-premise instance stopped serving entirely, returning 503 on every request. Restarting didn't help. A fresh image didn't help either. It had bricked itself with no recovery path short of hand-editing SQLite state on the volume.

## Root Cause

The health gate was a boolean, persisted to disk, set once the event queue fell behind and cleared only by the code path that drains it *successfully*. Events past `MAX_RETRY_COUNT` were deleted instead of delivered — so a queue emptied by deletion never ran the clearing path, and the flag stayed set through every restart.

## Solution

```python
# before: gate = self._backlog_flag           # latched, can stick
# after:
gate = (time.time() - self.oldest_unreported()) > GRACE_SECONDS
```

Derive the gate from the queue's own oldest-unreported timestamp instead of a flag. Add jitter to retries and quarantine (not delete) events the server keeps rejecting.

## 💡 Takeaway

- Prefer derived state to latched flags for anything gating service — a flag has failure modes a live query doesn't.
- "The queue is empty" and "the work was done" are different facts if your cleanup path can discard.
- A fail-closed gate must never be able to reject the exact request that would clear it.
