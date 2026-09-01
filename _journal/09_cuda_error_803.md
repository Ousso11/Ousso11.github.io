---
title: "CUDA Error 803 on the Newer Driver"
collection: journal
permalink: /journal/cuda-error-803-compat-lib/
excerpt: "An inference engine restart-looped on exactly the hosts with the most up-to-date drivers. The vLLM base image bundles a forward-compatibility libcuda ahead of the host driver on the loader path, and the loader picks the older one."
---

## Issue

One of our inference engines **restart-looped** on some GPU hosts and ran fine on others. CUDA initialisation failed with **error 803** (`system has unsupported display driver / cuda driver combination`).

The correlation was backwards from the obvious guess: it failed on hosts with **newer** drivers. Host driver 575 worked; host driver 580 did not.

## Root Cause

The vLLM base image bundles a **forward-compatibility `libcuda`** — a userspace driver shim (for driver 575) that lets a container built against a newer CUDA run on a host with an older driver. It is placed **ahead of the host driver on the loader path**, which is what makes forward compatibility work at all.

On a host that is *already newer* than the compat lib, that ordering inverts the intent: the loader picks the **older bundled shim** over the perfectly good host driver, and CUDA refuses the resulting mismatch. Error 803.

The mechanism designed to widen the range of supported drivers had become the thing narrowing it — and it fails on the hosts you would least expect, which is why the correlation looks inverted at first.

## Solution

**Remove the compat directory entirely** and pin the host library directories first on `LD_LIBRARY_PATH`.

The compat shim was buying nothing: CUDA 12.9 requires driver ≥ 525, which is true on every host in the fleet. Forward compatibility solves a problem we did not have, at the cost of a problem we did.

Verified: `torch.cuda.device_count() == 2` on the previously failing hosts.

## A Related Fix in the Same Area

The same engine also passed `data_parallel_size`, `max_num_batched_tokens` and `max_num_seqs` to a scorer class that **does not accept them** — a `TypeError` on first model load, meaning the failure surfaced at startup on the GPU box rather than at import or in CI.

Data parallelism moved to **process level**: one single-GPU replica per GPU behind an **nginx `least_conn`** load balancer, matching how our other engine already did it. Fewer framework-specific knobs, one scaling mechanism across both engines, and a failure mode that is a dead upstream rather than a constructor exception.

## 💡 Takeaway

- **Compatibility shims have a direction.** A forward-compat library ahead of the host driver helps old hosts and breaks new ones. If your floor is already satisfied everywhere, the shim is pure risk.
- **"Fails only on the newest machines" is a strong signal.** It points at version-ordering logic, not at the new machines.
- **Prefer the scaling mechanism you already operate.** Process-level replicas behind a load balancer were less clever than in-framework data parallelism and failed in a way the team already knew how to debug.
