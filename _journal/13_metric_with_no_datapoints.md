---
title: "[33] Ten Days of a Metric Nobody Was Writing"
collection: journal
order: 33
permalink: /journal/cloudwatch-iam-silent-denial/
excerpt: "A CloudWatch observability addon reported healthy while its node CPU metric had zero datapoints for ten days. The node IAM roles lacked PutMetricData, so every publish was denied silently — and a compliance control failed downstream because of it."
---

## Issue

The `amazon-cloudwatch-observability` addon was **active and healthy** on a production EKS cluster. `ContainerInsights.node_cpu_utilization` had **zero datapoints for 10+ days**.

Both statements were true at once. The addon reports on whether its pods are running, which they were. Nothing in its health model covers whether the data those pods emit is being accepted.

## Root Cause

Neither the system managed node group's IAM role nor the Karpenter node role carried **`CloudWatchAgentServerPolicy`**. The `cloudwatch-agent` DaemonSet's `PutMetricData` calls were being **denied — silently**.

IAM denials on a metrics publish path are close to invisible by design. The agent logs at a level nobody reads, retries, and continues. There is no alarm for "an alarm's input has stopped arriving," because the alarm has no way to distinguish "no data" from "nothing to report."

**Then it got worse downstream.** The empty cluster-aggregate metric left the cluster's node-CPU alarm in a permanent **`INSUFFICIENT_DATA`** state. Our compliance tooling treats a cluster-side CPU alarm that isn't reporting as absent, and **falls back to requiring per-instance `AWS/EC2` alarms** — so it flagged **every Karpenter GPU instance individually** as unmonitored.

The visible symptom was a compliance control failing across a fleet of GPU nodes. The cause was one missing IAM policy attachment, three layers away.

## Solution

Attach `CloudWatchAgentServerPolicy` to **both** node roles — the system MNG role and the Karpenter node role.

Two things about how it was rolled out:

**The live attach was done first, out of band.** IAM policy attachment takes effect on the next IMDS credential refresh, so metrics started publishing on the existing nodes **without rolling a single node** — no GPU workload disruption to fix a monitoring gap.

**Then the same change went into Terraform**, so the next `apply` is a no-op and — the part that actually matters — **future Karpenter node roles inherit it.** A live fix without the Terraform commit would have regressed the moment a new node pool was created, and regressed the same way: silently, for another ten days.

## 💡 Takeaway

- **"Addon healthy" answers a different question from "data arriving."** Liveness of a collector says nothing about acceptance of what it collects.
- **Alarm on staleness, not just on thresholds.** An alarm sitting in `INSUFFICIENT_DATA` is a failure that presents as silence.
- **IAM denials on write paths are the quietest failure in AWS.** Reads break loudly because something needs the answer. Writes just stop.
- **Fix live, then commit the same change.** The out-of-band attach bought immediate recovery with no node churn; the Terraform commit is what stops it recurring on the next node pool.
