---
title: "The GPU Node That Nothing Could Remove"
collection: journal
order: 28
permalink: /journal/stranded-burst-gpu-node/
excerpt: "A rollout surge pod landed on an on-demand GPU node and stayed pinned there for days at 0% utilisation, while reserved capacity sat idle. Three separate mechanisms — KEDA, Karpenter consolidation and the Kubernetes scheduler — each independently, correctly, refused to clean it up."
---

## Issue

A deployment rolled out with `maxSurge: 1` while every reserved GPU slot was still occupied by the outgoing pods. With nowhere to place the surge pod, the cluster autoscaler provisioned a **new on-demand GPU node** for it.

The pod then stayed on that node for **days**, at **0% GPU utilisation and under 1% CPU** — while **reserved, already-paid-for GPU capacity sat idle** the entire time.

We were paying on-demand rates for an idle accelerator in order to run a workload we had already prepaid for.

## Root Cause

Nothing was broken. Three mechanisms each correctly declined to act, and the intersection of three correct behaviours was an unbounded bill.

**KEDA was at minimum replicas.** From the autoscaler's perspective the deployment was already at its floor. Scaling down was not a decision it was declining to make — it was not a decision in its space at all.

**Karpenter's `consolidationPolicy: WhenEmpty` never fires on a non-empty node.** This is the crux. `WhenEmpty` means *literally zero pods*. The node had one running pod, so it was not a consolidation candidate — and since nothing would ever evict that pod, it would never become one. The node was permanently ineligible for the mechanism designed to reclaim it.

**The Kubernetes scheduler does not relocate running pods.** Scheduling is a one-time decision made at admission. There is no rebalancing loop; once bound to a node, a pod stays there until something external evicts it. "There is now a better place for this pod" is not an event the control plane reacts to.

Each component behaved exactly as configured. The system as a whole had **no path back** from burst capacity to reserved capacity — and burst capacity is the expensive kind.

This is the characteristic shape of a cloud cost incident: not a failure, but a missing return path.

## Solution

Three changes, so that burst capacity is **strictly additive and always returns**.

**1. Consolidation policy: `WhenEmptyOrUnderutilized`, with `consolidateAfter: 5m`.** Karpenter will now simulate whether a node's pods fit elsewhere in the cluster, evict and reschedule them onto free reserved slots, and reap the emptied node. The `consolidateAfter` delay stops it from thrashing on transient scheduling gaps.

Two safety properties matter as much as the fix:

- Reserved-capacity node groups sit **outside the cluster autoscaler's management by construction**, not by policy — so consolidation is structurally incapable of disturbing prepaid capacity.
- Node pools stay **partitioned per workload type**, so consolidation can repack a pod onto a free slot of the same kind but can never move it onto hardware meant for a different workload.

**2. Rollout strategy: `maxSurge: 0`, `maxUnavailable: 1`.** Rollouts now recycle a reserved slot **in place** — terminate one old pod, start one new pod in the slot it vacated — instead of demanding an additional slot that does not exist. This deliberately trades a brief capacity dip during a rollout for never provisioning hardware to hold a rollout artifact. On commodity CPU a surge pod is nearly free; on accelerators, `maxSurge: 1` is a request to buy the most expensive instance in the account.

**3. Scale-down tuned to actually fire.** Stabilisation window 300 s → 120 s, with a bounded drain rate of one pod per 60 s. Burst pods — and the nodes underneath them — now terminate minutes after traffic recedes rather than lingering. The rate limit keeps the scale-down itself from oscillating.

## 💡 Takeaway

- **Elastic capacity needs an explicit return path.** Scaling up is the half everyone tests. The expensive bugs live in the half that only runs when nobody is watching.
- **`WhenEmpty` is a far weaker guarantee than it reads as.** A single pod pins a node forever. `WhenEmptyOrUnderutilized` is what most people mean when they write `WhenEmpty`.
- **The scheduler has no memory and no regret.** Placement is decided once. Any notion of "this could now be packed better" has to come from a consolidation controller you explicitly configured.
- **Encode blast radius structurally, not in policy.** Reserved capacity being outside the autoscaler's reach *by construction* is a guarantee. Any setting that merely *should* protect it is a configuration away from not doing so.
