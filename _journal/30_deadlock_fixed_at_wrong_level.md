---
title: "[13] A Reentrant Lock Deadlock Froze the Proxy on Any Invalid Config Patch"
collection: journal
order: 13
permalink: /journal/split-the-locked-part-into-its-own-function/
excerpt: "A malformed config patch froze the entire proxy, permanently, because every request path calls Current() under the same lock. The first fix removed the self-deadlock and introduced a leaked write lock on both error paths — the common paths for a bad patch."
---

**The trick:** never hold a lock across a callback. Split the locked section into its own function whose only exit is a single `defer Unlock`, and run callbacks only after that function returns.

## Issue

A single invalid `PATCH /api/config` froze the whole Go proxy. Every request reads config through `Current()` under the same `sync.RWMutex`, so nothing recovered without a hard restart.

## Root Cause

`Reloader.Update` held `mu.Lock()` while calling subscriber callbacks, one of which called `Current()` — which takes `mu.RLock()`. `RWMutex` isn't reentrant. The first fix removed the deadlock by unlocking manually before the callback loop, but leaked the lock on both early-return error paths: validation failure and persist failure — exactly the paths a bad patch takes.

## Solution

```go
func (r *Reloader) applyUpdateLocked(patch Patch) (cfg Config, subs []Sub, err error) {
    r.mu.Lock()
    defer r.mu.Unlock()   // the only exit, on every path
    ...
    return cfg, subs, err
}
// caller runs subs' callbacks only after this returns, unlocked
```

## 💡 Takeaway

- Don't hand-roll unlocks before a return — split the locked work into its own function whose only exit is `defer`.
- Test a lock-holding function's error paths as hard as its success path.
- Reentrant deadlocks are usually two honest decisions meeting, not one obvious bug.
