---
title: "An asyncio.Lock Constructed at Import Time Binds to the Wrong Event Loop"
collection: journal
permalink: /journal/asyncio-lock-bound-to-wrong-event-loop/
excerpt: "A repository client's internal lock worked fine in isolated tests and deadlocked or raised confusingly under a real application's request lifecycle. It had been created once, at object construction, capturing whatever event loop happened to be running at that moment — not the one that would later serve requests."
---

**The trick:** never construct an `asyncio.Lock`, `Event`, `Queue`, or `Condition` eagerly in `__init__`. Create it lazily, on first use, inside the coroutine that will actually take it — that's the only point where the running event loop is guaranteed to be the right one.

## Issue

A client object's internal synchronization primitive worked correctly in small, isolated test cases and failed unpredictably once wired into a real application — sometimes a silent hang, sometimes a runtime error naming a completely different event loop than the one the request was running on.

## Root Cause

The client's lock was created in its constructor, at object-instantiation time. An `asyncio` synchronization primitive is bound to whichever event loop is running at the moment it's created — it is not a loop-agnostic value that can be freely awaited from anywhere later.

If the object is constructed before an application's real event loop has started — during module import, during dependency-injection setup, during a synchronous startup phase — the lock binds to whatever loop happens to be active at that instant, which in many frameworks is a transient bootstrap loop, or none at all until one is created implicitly. Every later `await lock.acquire()` from the application's actual request-handling loop is now operating on a primitive that belongs to a different loop entirely, which ranges from a confusing exception to a wait that never resolves, depending on exactly what the runtime does with a cross-loop primitive.

This is much easier to hit than it sounds, because plenty of test setups construct an object and immediately run it inside the same, single event loop for the whole test — so the bug is invisible until a real application's lifecycle, with distinct startup and serving phases, exposes the mismatch.

## Solution

Don't create the primitive in `__init__`. Store `None`, and create it lazily behind a small accessor called from within a coroutine:

```python
def _get_lock(self) -> asyncio.Lock:
    if self._lock is None:
        self._lock = asyncio.Lock()
    return self._lock
```

The first call happens from inside a coroutine that's actually running on the loop that will use it, which is the only correctness guarantee that matters. (This isn't itself concurrency-safe against two coroutines racing the first call — for most single-loop applications that's an acceptable, well-understood gap, and if you need to close it, guard the lazy-init with a plain attribute check under the GIL or a module-level `threading.Lock` rather than another `asyncio` primitive, which has exactly the same bootstrapping problem.)

## 💡 Takeaway

- **Anything from `asyncio` that's bound to a specific event loop must be created inside a coroutine running on that loop — never in `__init__`, never at module scope, never during synchronous startup.**
- **A test suite that runs everything inside one event loop for the whole test session cannot catch this class of bug.** It only appears once construction and use happen on genuinely different loops, which single-loop test setups paper over completely.
- **The fix is almost always lazy initialization behind an accessor**, not a smarter way to pass the primitive around — passing it around doesn't help if it was captured wrong in the first place.
