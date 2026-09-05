---
title: "Ten Times the GPU Work, and the Autoscaling Metric Never Noticed"
collection: journal
order: 3
permalink: /journal/token-aware-autoscaling-metric/
excerpt: "SageMaker's built-in autoscaling metric counts invocations per instance, not the compute behind them. A burst of long-generation requests looked identical to a burst of trivial ones, and the endpoint scaled three minutes too late every time — for exactly the traffic that mattered most."
---

**The trick:** autoscale on the actual driver of load, not on request count. A custom, high-resolution CloudWatch metric can turn a three-minute scaling decision into a twenty-second one.

## Issue

A GPU-backed LLM inference endpoint on SageMaker queued and ran at elevated latency for minutes during traffic bursts — even though autoscaling was configured correctly and did eventually add capacity.

## Root Cause

The built-in target-tracking metric, `SageMakerVariantInvocationsPerInstance`, counts requests, not compute. Ten short requests and ten requests each generating thousands of output tokens look identical to it. Combined with CloudWatch's 1-minute metric period and the consecutive breaches Application Auto Scaling requires before acting, a genuine GPU-saturating burst took roughly three minutes to trigger a scale-out — by which point the queue had already backed up twice over.

## Solution

Publish a custom high-resolution CloudWatch metric directly from the inference container — queue depth weighted by expected output length — at `StorageResolution: 1`, roughly every 10–15 seconds, and point the scaling policy at that instead of the built-in metric. Scaling decisions dropped to about twenty seconds. Because the metric now tracked actual load instead of raw count, instance-hours fell **9–32%** against the built-in policy by not over-provisioning for bursts of cheap requests.

## 💡 Takeaway

- A request-count metric and a compute-cost metric look identical until load becomes heterogeneous — which for LLM serving, it always eventually does.
- CloudWatch's metric period and evaluation-periods setting are latency baked into your scaling loop; a coarser metric than your traffic pattern needs will always lag it.
- The best autoscaling metric is rarely the one the platform ships by default — it's the one that actually predicts the resource you're trying to protect.
