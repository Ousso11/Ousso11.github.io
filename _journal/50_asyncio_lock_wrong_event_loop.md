---
title: "The Lock That Worked in Every Test and Hung in Production"
collection: journal
order: 11
permalink: /journal/asyncio-lock-bound-to-wrong-event-loop/
excerpt: "A repository client's internal lock worked fine in isolated tests and deadlocked or raised confusingly under a real application's request lifecycle. It had been created once, at object construction, capturing whatever event loop happened to be running at that moment — not the one that would later serve requests."
---

**The trick:** never construct an `asyncio.Lock`, `Event`, `Queue`, or `Condition` eagerly in `__init__`. Create it lazily, on first use, inside the coroutine that will actually take it.

## Issue

A client's internal lock worked correctly in isolated tests and failed unpredictably in the real application — sometimes a silent hang, sometimes an error naming an unrelated event loop.

## Root Cause

The lock was created in the constructor, binding to whatever event loop happened to be running at that instant — often a transient bootstrap loop, since construction ran before the application's real serving loop started. Every later `await` from the real loop operated on a primitive belonging to a different loop entirely.

## Solution

Create the lock lazily behind an accessor, called from inside a coroutine that's actually running on the loop that will use it — the only point where correctness is guaranteed.

## 💡 Takeaway

- Anything from `asyncio` bound to a specific loop must be created inside a coroutine on that loop — never in `__init__`, never at module scope.
- A test suite running everything in one event loop can't catch this class of bug at all.
- The fix is almost always lazy initialization, not a smarter way to pass the primitive around.
