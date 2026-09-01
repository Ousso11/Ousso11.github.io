---
title: "32.8 ms of Database Work on Every Compression Request"
collection: journal
permalink: /journal/telemetry-queue-serialization/
excerpt: "Telemetry was pinned to one core with writes behind a mutex. Three causes, measured on 10k queued events: a gate doing a full table scan under the store lock, one shared connection serializing readers behind writers, and a worker cap hiding a crash-recovery bug."
---

## Issue

A colleague observed the on-premise proxy's telemetry subsystem **pinned to a single core**, with writes queueing behind a mutex. On a product whose entire value proposition is lower latency, the measurement path had become the bottleneck.

Measured on 10,000 queued events, there were three independent causes. The first one was mine.

## Cause 1 — A Full Table Scan on the Request Path

The exposure gate I had added — the check that limits how much un-uploaded usage can accumulate before the proxy stops serving — computed its answer by **scanning every row and parsing each row's payload JSON**.

That ran **on every compression request**, and it ran **under the store lock**.

**32.8 ms per request.** Not on a background timer. On the hot path, holding the lock that every writer needs.

The fix is unglamorous and total: tokens became **a column, summed in SQL**, and the result is cached like the anchor value already was. The request path now does no database work in the common case.

```
32.8 ms → 0.0019 ms      (~17,000×)
```

The lesson is not "JSON parsing is slow." It is that a *policy check* quietly acquired the cost profile of a *report*, and nothing in the type system or the review flagged that a guard had become a scan.

## Cause 2 — One Connection, One Lock, Readers Blocking Writers

A single shared SQLite connection behind a single mutex meant **readers blocked writers** — which throws away the main reason to run SQLite in WAL mode at all. WAL exists so that a reader and a writer can proceed concurrently; a global lock in front of it reimposes exactly the serialization WAL removes.

Each thread now gets **its own connection**, and reads **drop the lock entirely**.

Under 8 writers plus a reader:

```
75 → 2,625 events/s       (35×)
contention  149× → 4×
```

The residual 4× is SQLite serializing *writers*, which is inherent to the engine and not worth fighting.

## Cause 3 — A Worker Cap Hiding a Correctness Bug

`WORKERS` was hard-capped at 1. Raising it caused **duplicate uploads**, and the cap had been left in place as the fix.

The actual defect: crash recovery released **every** claim, so a second process would steal a sibling's *in-flight* batch and upload it again. The cap was concealing a claim-ownership bug, not preventing a concurrency one.

Claims now carry a **timestamp**, and only genuinely abandoned ones — older than 4× the upload timeout — are reclaimed. `WORKERS` became environment-settable. **The default stays 1**, because the first two fixes removed the pressure that made anyone want to raise it, and a knob you don't need to turn is better than a knob you do.

## 💡 Takeaway

- **A guard is on the hot path by definition.** Anything that runs before every request has to be O(1)-ish; an exposure check that scans is a report that runs a million times a day.
- **Locks in front of WAL defeat WAL.** Per-thread connections and lock-free reads are the point of the mode.
- **A cap that prevents a bug is a bug with a lid on it.** `WORKERS=1` looked like tuning and was actually an unfixed claim-ownership defect.
- **Fix the cause, then leave the default alone.** Raising the ceiling was never the goal; removing the reason to raise it was.
