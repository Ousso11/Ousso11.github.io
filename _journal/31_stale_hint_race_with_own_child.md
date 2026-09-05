---
title: "The Fix That Raced Its Own Cleanup Out the Door"
collection: journal
order: 19
permalink: /journal/fix-reentered-by-your-own-subprocess/
excerpt: "Deleting a stale routing hint at launch fixed one bug and created another: the launcher re-execs itself as a daemon, so the child ran the same cleanup and deleted the hint the parent had just written. If a process re-execs itself, every startup side effect runs twice."
---

**The trick:** if a process re-execs itself, every "run once at startup" side effect runs twice, racing a child against its own parent — and explicit configuration should always outrank inferred configuration.

## Issue

A routing hint at `/tmp/context-gateway.upstream` could go stale after a `kill -9`, so we deleted it unconditionally at launch. Right after shipping that fix, launches started misrouting a *new* way — the hint file was empty immediately after being written.

## Root Cause

```go
// cmd/agent.go re-execs itself:
exec.Command(self, append(os.Args[1:], "--daemon")...)
```

The launcher re-execs itself with `--daemon`, running through the *same* startup code twice: once in the parent, which writes the hint, and once in the child moments later, which deleted it — indistinguishable, to the cleanup code, from a genuinely stale leftover.

## Solution

Gate the cleanup on `!daemonFlag`. Separately, move the explicit routing hint ahead of every inference heuristic in `autoDetectTargetURL` — it had been checked last, so any OpenAI-shaped request matched a heuristic before the explicit override was ever read.

## 💡 Takeaway

- Self-re-exec means every startup side effect runs twice, against a process racing itself.
- The obvious suspect for a race is another instance of the program — a self-re-exec is easy to forget entirely.
- Explicit configuration should always outrank inference, no matter how reliable the inference has been.
