---
title: "The Image That Didn't Exist Anymore, Still Running in Production"
collection: journal
permalink: /journal/image-tag-is-not-an-identity/
excerpt: "The Deployment spec showed exactly the image we intended. The pods had been serving an 18-hour-old build whose digest had already been deleted from the registry. A tag is a name; with imagePullPolicy IfNotPresent it is also a cache key."
---

**The trick:** pin deployments by content digest, never by a mutable tag — with `imagePullPolicy: IfNotPresent`, a tag is also a cache key, so a node can keep serving bytes the registry no longer has.

## Issue

An inference deployment was behaving like an older build. `kubectl get deploy -o yaml` showed exactly the image we intended to be running. Every part of the desired state was correct.

The pods had been serving an 18-hour-old build for 18 hours. Worse: the image digest they were actually running **had already been deleted from the registry**. There was no artefact left to inspect, and nothing in the Deployment could have told us.

## Root Cause

Three ordinary decisions composed into an invisible one.

**1. The registry repository was configured immutable.** That is a good default — it means a tag, once written, cannot be silently repointed at different bytes. But it also means "move the tag to the new build" is not an operation. It is a *delete* followed by a *create*.

**2. The Deployment was pinned by tag, not by digest.** So the thing we were promoting was a name, and the name had just been destroyed and recreated.

**3. `imagePullPolicy: IfNotPresent` caches on the tag string.** A node that already holds bytes under that tag does not consult the registry at all. It has something present under that name, so it uses it — including bytes that were deleted from the registry hours ago.

The result is two facts that look like one fact and are completely independent:

- what the Deployment *asks for* — visible in the spec, and it was right
- what the kubelet is *running* — visible only in `pod.status.containerStatuses[].imageID`, and it was wrong

There is a second, sharper hazard in the same mechanism: between the delete and the create, a live Deployment that needs to pull has nothing to pull. Immutability turns a routine promotion into a window where scaling up fails.

## Solution

**Pin the Deployments by `@sha256:` digest in the infrastructure code**, and make promotion an explicit three-step: build, tag a candidate, gate it, then set the digest.

This inverts the failure mode in a useful way:

- The identity in the spec is now the identity on the node. `IfNotPresent` caching becomes correct rather than dangerous, because the cache key *is* the content hash.
- **Rollback becomes a variable change**, not a registry operation. Both digests stay addressable no matter where the tags point, so going back is a one-line revert rather than a re-push.
- The delete-then-create window stops mattering, because nothing in the serving path resolves a tag.

The diagnostic habit that came out of it is worth as much as the fix: when an image question is in doubt, read `imageID` from the pod status, never the `image` field from the spec.

## 💡 Takeaway

- **A tag is a name. A digest is an identity.** Deploy the identity. Names are for humans choosing what to promote.
- **On an immutable registry, retagging is a deletion.** Immutability does not remove the mutable-pointer problem; it moves it and adds an outage window.
- **`IfNotPresent` is a cache, and its key is the string you gave it.** Any pull policy that can skip the registry makes "the spec is right" and "the node is right" separate claims requiring separate evidence.
- **When two sources of truth exist, find out which one the machine obeys.** We had been reading the one that was easy to read.
