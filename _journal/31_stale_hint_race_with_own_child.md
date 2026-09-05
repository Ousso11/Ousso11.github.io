---
title: "The Fix That Raced Its Own Cleanup Out the Door"
collection: journal
permalink: /journal/fix-reentered-by-your-own-subprocess/
excerpt: "Deleting a stale routing hint at launch fixed one bug and created another: the launcher re-execs itself as a daemon, so the child ran the same cleanup and deleted the hint the parent had just written. If a process re-execs itself, every startup side effect runs twice."
---

**The trick:** if a process re-execs itself, every "run once at startup" side effect runs twice, racing a child against its own parent. And explicit configuration should always outrank inferred configuration, no matter how reliable the inference has been.

## Issue

A routing hint file, used to remember which upstream a coding agent should talk to, could go stale if the process was killed with `SIGKILL`: the file survived on disk and a fresh launch inherited the wrong upstream. The fix was to delete the hint unconditionally at the top of the launch path.

Shortly after, launches started misrouting intermittently in a new way — the hint file was empty immediately after being written.

## Root Cause

The launcher does not run as a single process. It re-execs itself with a `--daemon` flag to become the long-running background process, and that re-exec **goes through the same launch code path**.

So the "delete the stale hint" fix ran twice: once in the parent when it first sets up routing and writes the hint, and once in the child, moments later, as the child re-entered the same startup logic on its way to becoming the daemon. The child had no way to know the hint it was deleting was fresh, just written by its own parent seconds earlier — it looked identical to a genuinely stale leftover from a killed process.

This is not a race between two unrelated processes. It's a race a fix introduced against **itself**, mediated by a parent/child relationship that the original bug report never had to think about.

A second, related bug lived in the same area: the hint was consulted as the *last* step in a chain of routing heuristics, so any request that superficially matched a well-known provider's shape got routed there first — a config that explicitly pointed elsewhere was silently overridden by inference.

## Solution

Gate the cleanup on whether this invocation is the daemon child. The parent, which is genuinely a fresh launch, still clears a truly stale hint; the child, which is a re-exec of a process that just wrote the hint moments ago, does not touch it. Both on-disk representations of the hint — a single-value form and a JSON map — are cleared together, so the two can no longer disagree with each other.

Separately, the explicit hint was moved to the front of the routing decision, ahead of every heuristic. An explicit pointer must always outrank inference, no matter how reliable the inference has been so far.

## 💡 Takeaway

- **If a process re-execs itself, every "run once at startup" side effect runs twice** — once in a process racing the other. Cleanup, initialization, and file deletion at launch all need to ask "am I the parent or the re-exec?" the moment self-re-exec exists anywhere in the codebase.
- **A fix for one race can create a second race with a process nobody suspected — its own child.** The obvious suspects for a race are other instances of the program; a self-re-exec is easy to forget because it doesn't look like concurrency from the outside.
- **Explicit configuration must always outrank inference.** A heuristic that's right 99% of the time is still a bug the moment it overrides a value the user or the config explicitly set.
