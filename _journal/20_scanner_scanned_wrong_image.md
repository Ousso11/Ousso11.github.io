---
title: "Five Fixes to One Line: The Security Gate That Scanned the Wrong Image"
collection: journal
permalink: /journal/gate-that-measured-the-wrong-artifact/
excerpt: "A container-configuration gate failed images that passed when scanned by tag, and passed images it had never looked at. It took five attempts, each of which fixed something real and left the gate still capable of reporting on an artefact nobody asked about."
---

**The trick:** derive the identifier any security gate measures from the artifact itself and never default it — and classify a tool crash as a crash, never as a finding against the thing it failed to scan.

## Issue

Our container-configuration scanner reported `FAIL` on images that passed cleanly when the same image was scanned by tag rather than by digest. Reports from that day carry a configuration section describing a **different image** from the other four sections of the same report.

A gate that can report on the wrong artefact is not a weak gate. It is not a gate at all — its `PASS` carries no information.

## Root Cause

The tool aborts with `FATAL: Manifest does not match provided manifest digest` when handed a `@sha256:` reference. Everything after that is the story of five fixes, each correct and each insufficient.

**Attempt 1 — retag the digest to a local alias.** `docker tag` requires the image to already be present locally, and the scan runs in a cold build container where it is not. The alias was never created, the digest was passed through unchanged, and the tool's crash was counted as a configuration finding *against the image*.

**Attempt 2 — pass an explicit tag in an environment variable.** It was exported in the shell of the person running the pipeline. The scan executes inside a hosted build service. `export` does not cross that boundary.

**Attempt 3 — plumb the variable through properly**, via the build service's environment override and the build spec. Correct, and it revealed the real defect underneath.

**Attempt 4 — the variable had a default.** It fell back to the pipeline's current tag. So scanning *any other* image by digest pointed two of the scanners at the requested image and the third at whatever the default named. Silent, plausible, and wrong in exactly the case where you most need the gate: verifying an artefact that is not the one you just built.

**Attempt 5 — two ordering bugs.** The digest-to-tag lookup ran *before* the override was consulted, so setting the variable did not prevent the `exit 1` whose error message instructed you to set it. And the registry host was assembled from account and region defaults rather than parsed out of the image reference — so scanning a digest belonging to a different account pointed the scanner at ours.

## Solution

Two properties, both structural:

**The scanned reference is derived from the digest, never defaulted.** If the resolution fails, the gate fails loudly. A fallback to something plausible is strictly worse than an error here, because it produces a result that looks like a measurement.

**A tool crash is classified as a crash.** It still fails the gate — we do not ship on an unscanned image — but it is no longer attributed to the image as a configuration finding. "We could not measure this" and "this measured badly" are different states and demand different responses.

## 💡 Takeaway

- **A gate that can measure the wrong artefact has no valid PASS.** Before trusting any check, ask what identifies the thing under test and whether that identifier can drift.
- **Never default the answer to "which artefact am I looking at".** Derive it, or fail. Defaults are for preferences, not for identity.
- **Distinguish "the tool failed" from "the subject failed".** Collapsing the two either hides breakage or invents findings; we managed both.
- **`export` stops at the process boundary you forgot about.** Anything that runs in a hosted builder needs its environment passed explicitly, and needs a test proving the value arrived.
- **Five fixes to one line is not incompetence, it is a layered bug.** Each attempt removed a real defect and exposed the next. The useful reflex is to keep asking "would this have caught the original symptom?" after every fix.
