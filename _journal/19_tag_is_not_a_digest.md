---
title: "The Image That Didn't Exist Anymore, Still Running in Production"
collection: journal
permalink: /journal/image-tag-is-not-an-identity/
excerpt: "The Deployment spec showed exactly the image we intended. The pods had been serving an 18-hour-old build whose digest had already been deleted from the registry. A tag is a name; with imagePullPolicy IfNotPresent it is also a cache key."
---

**The trick:** pin deployments by content digest, never by a mutable tag — with `imagePullPolicy: IfNotPresent`, a tag is also a cache key, so a node can keep serving bytes the registry no longer has.

## Issue

An inference deployment kept behaving like an older build. `kubectl get deploy -o yaml` showed exactly the image we intended. The pods had been serving an 18-hour-old build the entire time — and that build's digest had already been **deleted** from the registry.

## Root Cause

The registry was immutable, so moving a tag is delete-then-recreate, not repoint. The Deployment was pinned by tag, not digest. And `imagePullPolicy: IfNotPresent` caches on the tag *string* — a node already holding bytes under that name never re-checks the registry, even after the registry's copy is gone.

## Solution

Pin every Deployment by `@sha256:` digest in the infra code. Promotion becomes build → tag a candidate → gate → set the digest. Rollback is now a one-line variable change, not a registry operation.

## 💡 Takeaway

- A tag is a name; a digest is an identity. Deploy the identity.
- On an immutable registry, retagging is a deletion — there's a window where nothing is pullable.
- `IfNotPresent` is a cache keyed on the string you gave it. Check `pod.status.containerStatuses[].imageID`, never just the spec, when in doubt.
