---
title: "A Lockfile That Only Worked on macOS"
collection: journal
permalink: /journal/npm-lockfile-platform-optional-deps/
excerpt: "CI started failing on npm ci after a routine lockfile regeneration. Regenerating on macOS silently drops Linux-only optional dependencies, and re-running npm install on macOS will never add them back."
---

## Issue

CI broke on `npm ci` after two commits that did nothing more interesting than regenerate the lockfile. Locally, `npm install` worked, the build was clean, and the full test suite passed.

The failure was on the **Ubuntu runner**; the lockfile had been regenerated on **macOS**.

## Root Cause

Regenerating the lockfile on macOS **silently dropped Linux-platform optional entries** the Ubuntu runner needs — `@emnapi/runtime@1.10.0`/`1.11.0`, `p-retry@6.2.1`.

The part that makes this genuinely nasty:

> **Re-running `npm install` on macOS does not re-add them.** The local resolver only materialises optional dependencies whose platform matches the machine it is running on.

So the obvious remedy — "just regenerate it again" — reproduces the broken lockfile exactly, on every attempt, on the machine where you are trying to fix it. The lockfile is *correct for macOS* and structurally incapable of describing Linux, and nothing local will tell you that. The only environment that can observe the problem is the one that is failing.

## Solution

Reconstruct rather than regenerate:

1. Take the lockfile from **the last commit that passed CI**, immediately before the version bump.
2. `sed`-replace the package's own self-version reference to the new patch version.
3. `npm install --package-lock-only` to reconcile any range bumps **without** re-resolving the tree against the local platform.

Net diff: **1 insertion, 13 deletions** — mostly stale top-level self-version references. The Linux optional entries survived because they were never re-resolved. Locally: 260 tests pass, build clean.

The right long-term fix is to generate lockfiles in CI on the target platform rather than on a laptop — this was the surgical version, chosen to unblock a release rather than to relitigate the toolchain.

## 💡 Takeaway

- **A lockfile is platform-sensitive even though it claims to be a lock.** Optional dependencies are resolved against the machine doing the resolving.
- **"Regenerate and retry" can be a fixed point of the bug.** When the reproduction requires a platform you are not on, local iteration converges on the broken artifact.
- **Keep the last-known-good artifact identifiable.** Naming the exact commit that last passed CI turned an open-ended dependency debugging session into a three-step patch.
