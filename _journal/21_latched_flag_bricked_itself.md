---
title: "The Deployment That Bricked Itself Permanently"
collection: journal
permalink: /journal/derived-state-not-latched-flags/
excerpt: "An on-premise instance whose telemetry uploads kept failing began returning 503 and never stopped. Restarting did not help; shipping a new image did not help. The health anchor was a latched flag, and the only code that cleared it could no longer run."
---

**The trick:** replace a latched health/gate flag with a value derived live from the data it describes — a derived value can't get stuck set or stuck clear the way a flag can.

## Issue

An on-premise instance stopped serving. Every request returned 503. Restarting the container did not help. Shipping a new build did not help either.

The instance had bricked itself, permanently, with no recovery path that did not involve someone editing state on the volume by hand.

## Root Cause

Four defects in a durable event queue, compounding.

**The health anchor was a latched flag.** It was set when the queue fell too far behind, and cleared only by the code path that drains the queue successfully. But the failure handler *deleted* events once they passed a maximum retry count — so a queue that emptied **by deletion** rather than by delivery never ran the clearing path. The flag was also persisted to the data volume, which is why restart and upgrade were both no-ops: the stuck value was reloaded on boot.

**Every non-200 was treated as retryable.** A batch the server would reject forever — a bad credential, a schema the server no longer accepts — became a permanent head-of-line block. Nothing behind it could ever be delivered.

**A failed batch re-queued a successful one.** The claim was released as a single unit, so one tenant's rejected batch dragged another tenant's already-accepted events back into the queue.

**The retry backoff had no jitter.** Every instance in the fleet fails at the same instant when the far end goes down, and then retries in lockstep forever.

## Solution

**Derive the anchor from the queue instead of latching it.** The gate now asks the store a question — what is the oldest piece of unreported work? — rather than reading a flag somebody remembered to set.

That single change removes an entire category of bug. A value computed from durable state cannot get stuck set or stuck clear, needs no crash-recovery path, and cannot disagree with the data it describes.

**Classify upload outcomes into three states, not two:** OK, RETRY, and TERMINAL. Retryable client errors are exactly the ones that mean "try later" — request timeout and rate limiting. Everything else in the 4xx range means the server has made a decision and repeating the request is superstition.

**Quarantine what the server keeps rejecting.** Marked dead, retained on disk for inspection, excluded from claiming *and from the anchor* — because an undeliverable event must not hold a gate shut forever.

**Resolve each tenant's batch independently**, and add jitter to the backoff.

The subtle part came last. With a derived anchor, the queue's overflow policy became load-bearing. Evicting the *oldest* event on overflow would walk the anchor forward on every insert — so a saturated queue would mean the gate never closes, failing open in precisely the long-disconnection case the gate exists to catch. Overflow now rejects the **newest** event instead.

## 💡 Takeaway

- **Prefer derived state to latched flags for anything that gates serving.** A flag has two failure modes a query does not have: stuck set and stuck clear. And it needs its own recovery logic, which is itself code that can be wrong.
- **"The queue is empty" and "the work was done" are different facts.** If your cleanup path can empty a queue by discarding, every consumer of "empty" is now lying.
- **Retryable and terminal are not the same error class.** Treating all failures as retryable converts one bad message into a permanent outage.
- **Check what your eviction policy does to metrics derived from the queue.** Any statistic over "the oldest item" is silently redefined by the choice of what to drop under pressure.
- **A fail-closed path must never be able to reject the request that would clear it.** That is not a strict gate, it is a deadlock.
