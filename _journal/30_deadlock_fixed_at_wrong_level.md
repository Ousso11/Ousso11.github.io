---
title: "One Bad Config Patch Froze the Proxy, Forever"
collection: journal
permalink: /journal/split-the-locked-part-into-its-own-function/
excerpt: "A malformed config patch froze the entire proxy, permanently, because every request path calls Current() under the same lock. The first fix removed the self-deadlock and introduced a leaked write lock on both error paths — the common paths for a bad patch."
---

**The trick:** never hold a lock across a callback. Split the locked section into its own function whose only exit is a single `defer Unlock`, and run callbacks only after that function returns.

## Issue

Submitting an invalid configuration patch to the running proxy froze it completely. Not just the reload — every subsequent request, forever, because every request path reads the current configuration under the same lock the reload was holding.

## Root Cause

The update routine held the write lock for the entire operation, including the loop that notifies subscribers of the new config. One of those subscriber callbacks calls the read-accessor for the current config, which takes the read lock. Go's `RWMutex` is not reentrant, so a writer calling back into a reader of the same lock deadlocks itself, unconditionally, on every update.

## Solution — attempt one, wrong level

The first fix removed the `defer Unlock()` and instead unlocked manually, right before the callback loop, after copying the subscriber list. That solved the reentrancy: the callbacks now ran unlocked.

It introduced a new bug on the paths that mattered most. There were two early returns above the callback loop — patch validation failure and persist-to-disk failure — and both of them exited **before** reaching the manual unlock. Since a bad patch is exactly what triggers validation failure, the *common* path for the exact scenario we were fixing now leaked the write lock and froze the proxy anyway, just via a different mechanism.

## Solution — attempt two, right level

Split the function. A private function owns the entire critical section and has exactly one exit: `defer Unlock()`, guaranteed on every return including both error paths. It computes the new state and returns it, along with the subscriber list and any error — but it never calls a callback. The public function calls the private one, and only *after* it returns does it run the callback loop, fully outside the lock.

No return path inside the locked function can leak the lock, because there is only one way out of it. And no callback can re-enter the lock, because none of them execute while it's held.

## 💡 Takeaway

- **When a lock must not be held across a callback, don't hand-roll the unlock.** A manual `Unlock()` placed before a `return` is safe only until someone adds a second early return — which validation code does constantly.
- **Split the locked work into its own function whose only exit is a `defer`.** Return the follow-up work (callbacks, notifications, I/O) to the caller to run unlocked. This makes "cannot leak the lock" a property of the function's shape, not of everyone remembering to unlock correctly.
- **Reentrant deadlocks are usually two honest decisions meeting.** Nobody wrote "call back into my own lock" on purpose — one function held a lock for safety, another called a public accessor for convenience, and the two agreed to deadlock the first time they met.
- **Test the error paths of a lock-holding function as carefully as the success path.** The bug that survived here was invisible on a valid patch and guaranteed on an invalid one — which is precisely the input a fuzzing or property test should throw at it.
