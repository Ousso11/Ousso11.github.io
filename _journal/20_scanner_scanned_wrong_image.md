---
title: "The Security Gate That Never Once Looked at Our Image"
collection: journal
permalink: /journal/gate-that-measured-the-wrong-artifact/
excerpt: "A container-configuration gate failed images that passed when scanned by tag, and passed images it had never looked at. It took five attempts, each of which fixed something real and left the gate still capable of reporting on an artefact nobody asked about."
---

**The trick:** derive the identifier a security gate measures from the artifact itself, and never default it — and classify a tool crash as a crash, never as a finding against the thing it failed to scan.

## Issue

Our container-config scanner reported `FAIL` on images that passed cleanly when scanned by tag instead of digest. Reports from the same day described a **different image** in one section than in the other four.

## Root Cause

The scanner crashes on a digest reference. Five successive fixes each solved a real problem and left one gap open: a local-alias workaround that silently no-op'd in a cold build container; an env var exported on the wrong machine; a variable that defaulted to the wrong image when unset; and a lookup order bug that made the fallback fire before the real value was even checked. Every version could still, in some case, measure something other than the image being shipped.

## Solution

Derive the scanned reference from the digest with no default — fail loudly if it can't be resolved. Separately, classify a tool crash as a crash, not as a configuration finding against the image; the two need different responses.

## 💡 Takeaway

- A gate that can measure the wrong artifact has no valid PASS. Verify what it's actually looking at before trusting the result.
- Never default the answer to "which artifact is this." Derive it, or fail.
- Five fixes to one bug usually means a layered bug — after each fix, ask whether it would have caught the *original* symptom.
