---
title: "A Socket-Level Timeout Failed to Bound a Slow-Drip Request"
collection: journal
order: 14
permalink: /journal/socket-timeout-vs-wall-clock-deadline/
excerpt: "A hot-path call configured with a timeout could hang indefinitely. A server dribbling one byte before each read-timeout window resets the clock every time — the timeout never fires, because it was never bounding the thing that needed bounding."
---

**The trick:** a socket timeout bounds one read, not one request. For a real wall-clock deadline on a blocking call you can't cancel, run it on a thread and abandon the thread if the deadline passes.

## Issue

A call on a latency-sensitive path, configured with an explicit timeout, could hang far longer than that timeout — in the worst case, indefinitely.

## Root Cause

The timeout bounded each individual socket read, not the request as a whole. A server dribbling one byte just under each read window resets the clock every time — the request runs forever while the "timeout" never once fires, because it was watching the wrong thing.

## Solution

Run the blocking call on a background thread and enforce a genuine wall-clock deadline against it. Past the deadline, abandon the thread and fail safe — there's no way to forcibly cancel the underlying blocking call.

## 💡 Takeaway

- A socket timeout bounds a single `recv`, not a request. Anything needing "finishes or dies in N seconds" needs its own wall-clock deadline.
- Size caps and time caps are different protections; neither implies the other.
- When you can't cancel a blocking call, abandon the thread running it rather than waiting past the deadline.
