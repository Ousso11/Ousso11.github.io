---
title: "The Fix That Raced Its Own Cleanup Out the Door"
collection: journal
permalink: /journal/fix-reentered-by-your-own-subprocess/
excerpt: "Deleting a stale routing hint at launch fixed one bug and created another: the launcher re-execs itself as a daemon, so the child ran the same cleanup and deleted the hint the parent had just written. If a process re-execs itself, every startup side effect runs twice."
---

**The trick:** if a process re-execs itself, every "run once at startup" side effect runs twice, racing a child against its own parent — and explicit configuration should always outrank inferred configuration.

## Issue

A routing hint file could go stale after a killed process, so we deleted it unconditionally at launch. Shortly after shipping that fix, launches started misrouting a new way — the hint was empty right after being written.

## Root Cause

The launcher re-execs itself with a daemon flag, running through the same startup code twice: once in the parent, which writes the hint, and once in the child moments later, which deleted it — indistinguishable, to the cleanup code, from a genuinely stale leftover.

## Solution

Gate the cleanup on whether this invocation is the daemon child. Separately, move the explicit routing hint ahead of every inference heuristic in priority.

## 💡 Takeaway

- Self-re-exec means every startup side effect runs twice, against a process racing itself.
- The obvious suspect for a race is another instance of the program — a self-re-exec is easy to forget entirely.
- Explicit configuration should always outrank inference, no matter how reliable the inference has been.
