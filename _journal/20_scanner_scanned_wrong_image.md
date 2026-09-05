---
title: "[15] A Misconfigured Security Scanner Never Scanned the Intended Image"
collection: journal
order: 15
permalink: /journal/gate-that-measured-the-wrong-artifact/
excerpt: "A container-configuration gate failed images that passed when scanned by tag, and passed images it had never looked at. It took five attempts, each of which fixed something real and left the gate still capable of reporting on an artefact nobody asked about."
---

**The trick:** derive the identifier a security gate measures from the artifact itself, and never default it — and classify a tool crash as a crash, never as a finding against the thing it failed to scan.

## Issue

Our Dockle-based config gate reported `FAIL` on images that passed cleanly when scanned by tag instead of digest. Reports from the same day described a **different image** in the config section than in the other four.

## Root Cause

Dockle aborts with `FATAL: Manifest does not match provided manifest digest` on a `@sha256:` ref. Five successive fixes each solved a real bug and left one gap: a `docker tag` alias that silently no-op'd because the image wasn't local yet; an env var exported on the CLI operator's laptop instead of the CI runner; a `DOCKLE_IMAGE_REF` that defaulted to `$TAG` when unset, so it scanned the *previous* build; and a lookup-order bug where the fallback ran before the real value was checked.

## Solution

```bash
DOCKLE_IMAGE_REF="$(aws ecr batch-get-image --image-ids imageDigest="$DIGEST" \
  --query 'images[0].imageId.imageTag' --output text)" || exit 1
```

Resolve the ref from the digest with no default — hard-fail if resolution fails. Classify a Dockle crash as a crash, not a configuration finding against the image under test.

## 💡 Takeaway

- A gate that can measure the wrong artifact has no valid PASS. Verify what it's actually scanning.
- Never default the answer to "which artifact is this." Derive it, or fail loudly.
- Five fixes to one bug usually means a layered bug — after each fix, ask whether it would've caught the *original* symptom.
