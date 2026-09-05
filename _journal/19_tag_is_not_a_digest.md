---
title: "Tag-Based Image Pinning Let Kubernetes Serve a Deleted Build"
collection: journal
order: 11
permalink: /journal/image-tag-is-not-an-identity/
excerpt: "The Deployment spec showed exactly the image we intended. The pods had been serving an 18-hour-old build whose digest had already been deleted from the registry. A tag is a name; with imagePullPolicy IfNotPresent it is also a cache key."
---

**The trick:** pin Kubernetes Deployments by content digest, never by a mutable tag — with `imagePullPolicy: IfNotPresent`, a tag is also a cache key, so a node can keep serving bytes the registry no longer has.

## Issue

An inference Deployment kept behaving like an older build. `kubectl get deploy -o yaml` showed exactly the image we intended. The pods had been serving an 18-hour-old build for those 18 hours — and that build's digest had already been **deleted** from ECR.

## Root Cause

ECR was set to immutable tags, so moving a tag is delete-then-recreate, not repoint. The Deployment pinned `image: registry/engine:prod`, not a digest. And `imagePullPolicy: IfNotPresent` caches on the tag *string* — a node already holding bytes under that name never re-checks the registry, even after the registry's copy is gone. `pod.status.containerStatuses[].imageID` was the only place the drift was visible.

## Solution

```yaml
image: registry/engine@sha256:9f2c1e...   # digest, not tag
```

Promotion became: build → tag a candidate → gate → set the digest in Terraform. Rollback is now a one-line variable change, not a registry operation.

## 💡 Takeaway

- A tag is a name; a digest is an identity. Deploy the identity.
- On an immutable registry, retagging is a deletion — there's a window where nothing is pullable.
- Check `imageID` on the running pod, never just the spec, when a rollout looks stale.
