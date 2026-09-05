---
title: "[37] Six-Hour Benchmark Sweeps That Survive a Crash"
collection: journal
order: 37
permalink: /journal/atomic-resume-benchmark-sweeps/
excerpt: "A multi-config benchmark sweep runs for hours and dies for reasons that have nothing to do with the experiment. Writing results atomically per question — rather than per run — turned three 20-minute stalls from lost days into lost minutes."
---

## Issue

A full sweep across configurations and 200 questions runs for hours, and machines do not reliably stay up for hours. Networks blip, a provider rate-limits, a process gets killed, a question hangs.

Under the original design — write the artifact when the run completes — **any failure costs the entire run.** The practical consequence is worse than the wasted compute: it changes what you are willing to measure. Nobody starts a six-hour sweep at 4pm if a hiccup at hour five loses everything, so sweeps get smaller and results get noisier for reasons that have nothing to do with the science.

## Solution

**Write after every completed question, atomically.**

The runner now writes the per-config JSON artifact after each question finishes — `partial=true` while in flight, and a full graded artifact on success. Atomic write (write-temp-then-rename) matters: a crash *during* the write must not leave a truncated JSON file, which would turn a recoverable failure into an unrecoverable one and be strictly worse than not checkpointing.

**Skip on the way back in.** On startup the runner loads any prior artifact, keeps already-completed non-errored rows, and skips those sample IDs. Two properties are load-bearing:

- **non-errored** — a failed row is retried rather than cached as a failure, so a transient provider error doesn't get frozen into the results;
- **by sample ID** — not by count, so resumption is correct even if question order changes between runs.

**`--resume <timestamp_or_path>`** reuses a prior timestamped output directory instead of creating a new one, so a killed multi-config sweep restarts without redoing finished configs.

Verified end-to-end on n=2 × 2 configs: the first pass writes both artifacts incrementally; the second pass with `--resume` skips both configs in 0.2 s each.

## Why It Mattered

The n=200 head-to-head run that produced our published comparison **hit three separate 20+ minute stalls**. Every one of them was recovered without losing graded work. Without resume, that experiment would have been attempted, abandoned, and re-scoped to a smaller sample — and a smaller sample was the difference between a result and a noise band.

The same principle shaped the rest of the harness: **judging runs from stored model outputs, not live ones**, so a rubric change costs a judging pass instead of re-compressing and re-answering everything.

## 💡 Takeaway

- **Checkpoint at the unit of work, not the unit of run.** A question takes seconds; a sweep takes hours. Write at the granularity you can afford to lose.
- **Atomic or it's not a checkpoint.** A partially-written artifact is worse than none, because it corrupts the recovery path too.
- **Resume must distinguish "done" from "failed."** Caching failures is how a transient outage becomes a permanent result.
- **Recoverability changes what you are willing to measure.** The technical win is saved compute; the real win is that larger, longer, more honest experiments become affordable to attempt.
