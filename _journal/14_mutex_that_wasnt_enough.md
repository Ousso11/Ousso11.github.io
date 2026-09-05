---
title: "A Mutex That Wasn't Enough: Racing on a Global Logger"
collection: journal
order: 34
permalink: /journal/sync-once-global-logger/
excerpt: "Go's race detector kept failing CI on the global logger even after the write was put under a mutex. A mutex only protects the accesses that take it — and every log.Info() call in the codebase read the same variable without one."
---

## Issue

The Go race detector failed CI intermittently on the gateway's global logger. The first fix was the obvious one: put the write to `log.Logger` under a mutex in `Global()`.

The race detector kept failing.

## Root Cause

The mutex was doing exactly what a mutex does, and it was not enough:

- **`Global()` writes** `log.Logger` **under the mutex.**
- **Every dashboard goroutine reads** `log.Logger` **via `log.Info()` — without taking it.**

A mutex creates mutual exclusion only among the accesses that participate. Guarding one side of a write/read pair guards nothing; it just makes the writer feel safe. The data race between `Global()`'s write and the hundreds of unsynchronised `log.Info()` reads was untouched.

The honest options were: take the lock on every read — putting a mutex acquisition in front of every log line in the process, on the hot path, for a value that changes at most once — or remove the write/read race by construction.

It surfaced under **parallel test gateways** rather than in production, because that is where multiple gateway instances initialise concurrently in a single process. Tests were the only workload that ever raced.

## Solution

**`sync.Once`.**

```go
var once sync.Once

func Global() *Logger {
    once.Do(func() { /* build and assign exactly once */ })
    return logger
}
```

The global logger is now set **exactly once per process**. There is no write-after-read to protect against, so readers need no synchronisation at all and the hot path stays free of lock acquisition. `sync.Once` also provides the happens-before edge that makes the initialised value visible to every subsequent reader — which a plain "check if nil, then assign" would not.

## 💡 Takeaway

- **A mutex is a protocol, not a property.** It protects a variable only if *every* access takes it. Locking the writer alone is a very convincing no-op.
- **Prefer eliminating the race to synchronising it.** Write-once initialisation removed the shared mutable state instead of arbitrating access to it — and cost nothing per read.
- **Parallel tests are a legitimate concurrency workload.** This race only ever appeared in CI. That does not make it a test bug.
- **Run the race detector in CI.** Nothing here was reachable by reading the code; the detector found it, twice.
