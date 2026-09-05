---
title: "The Build Failed and Shipped Anyway: set -e One Line Before the Re-tag"
collection: journal
permalink: /journal/set-e-and-the-and-and-idiom/
excerpt: "A build reported failure and pushed the un-flattened image to the registry regardless. Two uses of the [ -n \"$x\" ] && echo idiom returned non-zero on an empty value, and under set -e the script died just before the step that mattered."
---

**The trick:** end any `[ cond ] && cmd` idiom with `|| :` when it's the last statement of a script under `set -e` — otherwise a false test aborts the whole script, silently, one line before the step that mattered.

## Issue

A pipeline stage failed in CI. It also **published its output** — the wrong output. The registry received the raw, un-flattened image under the final tag, and the build reported failure afterwards.

Both halves are bad, and the combination is worse than either: a red build that changed production.

## Root Cause

The stage rebuilds a many-layered image as a single layer, which means reconstructing the configuration by hand — reading environment, working directory, exposed ports, entrypoint and command out of the image metadata and re-emitting them into a synthetic Dockerfile.

Two of those lines used a very common shell idiom:

```bash
[ -n "$WORKDIR" ] && echo "WORKDIR $WORKDIR" >> "$dockerfile"
```

When the value is empty, the test is false, the `&&` chain short-circuits, and **the whole statement returns non-zero**. Under `set -e`, that terminates the script.

The termination point was one line before the re-tag. So the flattening work was abandoned mid-way, the earlier, un-flattened image was still sitting under the tag the pipeline pushes, and the push proceeded from a different stage.

The idiom is fine in the middle of a script where a later command overwrites the status. It is a trap as the last statement of a script, a function, or a loop body — precisely where "optionally do this thing" tends to be written.

## Solution

```bash
[ -n "$WORKDIR" ] && echo "WORKDIR $WORKDIR" >> "$dockerfile" || :
```

Two characters. `|| :` makes the failure branch succeed explicitly, which is what the author meant all along.

The ordering hazard got fixed too. Any step whose *commit* — re-tag, push, promote, publish — happens after fallible cosmetic work has the failure mode we hit: partial work, then an exit, then a publication of whatever was there before. The commit step now runs from a verified artefact or not at all.

The same file taught a second lesson worth recording. `docker import` strips **both** `ENTRYPOINT` and `CMD`. The entrypoint here ends with `exec "$@"`, so `CMD` is load-bearing arguments, not a default — losing it produces a container that starts, receives nothing to exec, and restart-loops. Metadata you did not know was semantic is still semantic.

## 💡 Takeaway

- **`[ cond ] && cmd` as the final statement under `set -e` is a landmine.** Terminate it with `|| :` or write it as a real `if`.
- **`set -e` turns "this optional step did nothing" into "the program is over".** The exit status of a conditional is not the exit status of a failure.
- **Order the irreversible step last, and gate it on the work having actually happened.** A stage that can die between "produce" and "publish" will eventually publish something it did not produce.
- **A red build that still changed production is the worst possible outcome.** It removes the one signal you would have used to know not to look.
