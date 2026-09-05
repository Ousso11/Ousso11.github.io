---
title: "[25] The Autoscaling Study That Crowned the Wrong Winner"
collection: journal
order: 25
permalink: /journal/score-the-decision-not-the-launch/
excerpt: "A GPU autoscaling experiment ranked a policy that refused to scale as second-best overall. Four scoring bugs — the headline one being that we measured instance availability instead of the scaling decision, inflating a 33× gap into 7×."
---

## Issue

An unattended autoscaling study compared scaling metrics for a GPU fleet across traffic mixes, ranked them on a composite of cost and SLO breach, and produced a winner.

The winner was wrong. Worse, the runner-up — the invocations-based metric — **never left one instance**, breached its SLO by 35.6% on heavy traffic, and still placed 2nd overall.

When a configuration that refuses to do the job scores well, the problem is the scorer.

## Root Cause

Four bugs, all in measurement rather than in the system being measured.

**1. We measured availability, not the decision.**

`scale_out_lag` was computed on `CurrentInstanceCount` — when an instance became *available*. That number includes instance provisioning, a multi-gigabyte image pull, and model server startup: three things a scaling metric has no control over. It should measure `DesiredInstanceCount` — the moment the policy *decided*.

Recomputed on the existing trial data, this is not a small correction:

| Metric | Reported | Actual decision latency |
|---|---|---|
| Token-throughput (high-resolution) | 66 s | **5 s** |
| Invocations | 451 s | **167 s** |

A **33× gap in decision latency was being displayed as 7×** — the provisioning constant dominated both numbers and compressed the thing we were trying to distinguish.

**2. The feasibility gate had been calibrated against the wrong quantity.** `max_lag_s = 30` was set against availability, so once lag was measured correctly *all 8 trials* came back infeasible and the gate silently did nothing. Re-set to 120 s against the decision, it admits the concurrency metric (66 s) and rejects the invocations metric (167 s) — which is the discrimination the gate exists to provide.

**3. Ranking mixed infeasible configs in with feasible ones.** Cost is a third of the composite, so **a config that refuses to scale scores well for being cheap.** That is the entire explanation for the invocations metric placing 2nd. Feasibility now sorts first, and cost only breaks ties among configurations that actually work.

**4. Family selection used the best traffic mix.** A family was picked by its strongest mix, which discards the whole point of crossing traffic patterns. Selecting on the **worst** mix drops invocations (63.0% breach on mid) below concurrency (19.6% on heavy).

And one that invalidated more than it looked like: the observer recorded `describe_scaling_activities` **unfiltered**, so **49 of 50 activities in the study came from earlier runs with different policies and caps.** Anything keyed on them was reading someone else's experiment. Fixed with an `in_run_activities()` window.

## Solution

Measure `DesiredInstanceCount` for the decision, keep availability as a separate metric (still worth knowing — it is just not the policy's fault), recalibrate the gate against the corrected quantity, sort feasible-first, select on the worst mix, and scope activity queries to the run.

The trial data was committed alongside the fix, so the recompute is reproducible rather than asserted.

## 💡 Takeaway

- **Measure the decision you are evaluating, not the outcome it is embedded in.** A large constant shared by every arm doesn't cancel out — it hides the differences you are paying to measure.
- **A ranking that can reward doing nothing is broken.** Feasibility is a filter, never a term in a weighted sum.
- **Selecting on the best case defeats the purpose of testing multiple cases.** Worst-case selection is what "robust across traffic" actually means.
- **Recalibrate every threshold that was tuned against a number you just corrected.** Fixing the metric silently disabled the gate that depended on it.
