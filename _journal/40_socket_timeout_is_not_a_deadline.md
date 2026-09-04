---
title: "A Socket Timeout Bounds a Read, Not a Request"
collection: journal
permalink: /journal/socket-timeout-vs-wall-clock-deadline/
excerpt: "A hot-path call configured with a timeout could hang indefinitely. A server dribbling one byte before each read-timeout window resets the clock every time — the timeout never fires, because it was never bounding the thing that needed bounding."
---

## Issue

A call on a latency-sensitive hot path, configured with an explicit timeout, could hang far longer than that timeout — in the worst observed case, indefinitely.

## Root Cause

The timeout in question bounds a single low-level socket read, not the request as a whole. A server that sends data slowly — one byte, then a pause, then another byte, each pause just under the timeout window — never triggers the timeout, because every individual read completes within its allotted window. The overall request can run forever this way, one byte at a time, with the client's "timeout" never once firing, because it was watching the wrong thing.

A related bound made it worse. A separate cap on how many bytes a client would read protected against unbounded memory growth from a huge response, but it bounded *size*, not *time* — a call could still hang forever on a response that trickled in slowly, well under the size cap, for as long as the far end chose to keep sending.

## Solution

Run the blocking call on a background thread and wait on it with a genuine wall-clock deadline. If the deadline passes before the thread finishes, the caller gives up and fails safe, abandoning the thread rather than waiting on it further — there is no way to forcibly cancel a blocking call in the underlying library, so abandonment is the only available option. The deadline is set as a multiple of the configured timeout, giving the underlying call room to legitimately finish slow-but-healthy requests while still bounding the pathological case. Reading is done incrementally, in fixed-size increments, with an explicit cap on total accumulated bytes — solving the size problem and the time problem as two separate, explicit bounds rather than assuming one implies the other.

## 💡 Takeaway

- **A socket-level timeout bounds one read, not one request.** Anything that needs "this call finishes or dies within N seconds" needs its own wall-clock deadline, independent of and layered on top of whatever the transport already provides.
- **Size caps and time caps are different protections and neither implies the other.** A response can be small and slow, or fast and huge; you need to bound both explicitly.
- **When the only tool available for a blocking call you cannot cancel is a background thread, abandon it rather than waiting on it past the deadline.** It's not elegant, but it's the only way to make "this returns in bounded time" true when the underlying primitive offers no cancellation.
