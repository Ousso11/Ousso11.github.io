---
title: "Starting a Second Agent Killed the First One"
collection: journal
order: 35
permalink: /journal/stale-lock-file-pid-check/
excerpt: "Opening a coding agent in a second terminal caused the session in the first terminal to exit. Our launcher deleted every IDE lock file on startup — including the ones belonging to running sessions."
---

## Issue

Run the gateway's agent launcher in Terminal 1, work happily. Open Terminal 2 and start a second session. **Terminal 1's session exits.**

Reported as a gateway bug, and reasonably so — it only happened when running through our launcher. It was a launcher bug, and a self-inflicted one.

## Root Cause

On startup, the launcher **deleted all Claude IDE lock files**.

The intent was sound: a crashed session leaves a lock file behind, and a stale lock blocks the next start. Clearing them made startup reliable — as long as exactly one session ever existed.

But the agent **watches its own lock file** and treats removal as a signal to shut down, which is the correct behaviour for it: a vanished lock means something else has claimed the session. So starting instance #2 deleted instance #1's live lock, and instance #1 did precisely what it was designed to do.

The deletion could not distinguish **"left over from a crash"** from **"held by a process running right now"** — because it never asked. Both look identical on disk. The only difference is whether the owning process still exists, and that information was available the whole time.

## Solution

**Check whether the owning PID is still alive before removing a lock file.** Only genuinely stale locks — those whose process is gone — get cleaned up.

Live sessions keep their locks, so multiple terminals can run agents simultaneously without interfering, and crash recovery still works because a dead process's PID does not resolve.

## 💡 Takeaway

- **A lock file without liveness information is just a file.** The PID is what makes it a lock; checking it is what makes cleanup safe.
- **"Clear all state on startup" scales to exactly one instance.** It is a very common shape of fix that works perfectly until the second user, and then fails in a way that looks like someone else's bug.
- **Interpret bug reports structurally.** "The agent exits randomly" and "the agent exits when I open a second terminal" are the same defect; only the second one is debuggable, and getting there took asking what else was running.
