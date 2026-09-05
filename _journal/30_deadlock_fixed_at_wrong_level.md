---
title: "One Bad Config Patch Froze the Proxy, Forever"
collection: journal
permalink: /journal/split-the-locked-part-into-its-own-function/
excerpt: "A malformed config patch froze the entire proxy, permanently, because every request path calls Current() under the same lock. The first fix removed the self-deadlock and introduced a leaked write lock on both error paths — the common paths for a bad patch."
---

**The trick:** never hold a lock across a callback. Split the locked section into its own function whose only exit is a single `defer Unlock`, and run callbacks only after that function returns.

## Issue

A single invalid config patch froze the whole proxy. Every request reads the config under the same lock the reload was holding, so nothing recovered without a restart.

## Root Cause

The reload routine held the write lock while notifying subscribers, one of which read the config back under the read lock — a self-deadlock on Go's non-reentrant `RWMutex`. The first fix unlocked manually before the callback loop, which fixed reentrancy but leaked the lock on both early-return error paths — exactly the paths a bad patch takes.

## Solution

Split the function: a private function owns the whole critical section with one exit, `defer Unlock`, guaranteed on every return. It returns the new state and subscriber list; only the caller, unlocked, runs the callbacks.

## 💡 Takeaway

- Don't hand-roll unlocks before a return — split the locked work into its own function whose only exit is `defer`.
- Test a lock-holding function's error paths as hard as its success path.
- Reentrant deadlocks are usually two honest decisions meeting, not one obvious bug.
