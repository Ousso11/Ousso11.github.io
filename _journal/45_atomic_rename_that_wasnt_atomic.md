---
title: "The Atomic Write That Wasn't, and Only on Linux"
collection: journal
permalink: /journal/temp-file-rename-across-filesystems/
excerpt: "A config persistence routine used the standard temp-file-then-rename pattern for an atomic write. On Linux, where the system temp directory is routinely a different filesystem from the target, the rename fails outright — and the failure path was the one that caused the deadlock in an earlier story."
---

## Issue

A configuration persistence routine used the standard pattern for a safe, atomic write: create a temporary file, write the new contents, then rename it into place. Rename is atomic on a POSIX filesystem, so a crash mid-write can never leave a half-written config behind. On Linux, this "atomic" write was in fact the routine's most reliable way to fail.

## Root Cause

The temporary file was created in the system's default temporary directory. On a great many Linux systems, that directory is mounted as its own filesystem — commonly an in-memory one — separate from wherever the target configuration file actually lives. A rename across two different filesystems is not a rename at all as far as the underlying system call is concerned; it fails outright with a cross-device-link error, because a rename can only relink an entry within a single filesystem, never copy bytes between two.

So the "atomic persist" path was silently filesystem-dependent: harmless on a system where temp and target happen to share a filesystem, and a guaranteed failure everywhere they don't — which describes a large share of real Linux deployments. This was the same code path implicated in an earlier deadlock story: the persist failure was one of the two error returns that could leak a held lock, so this bug and that one compounded.

A second defect shared the same file: persisting the configuration wrote out environment-variable references in their **already-expanded** form — a secret reference in the source configuration was written back to disk as the literal secret value, duplicated across every section that had referenced it.

## Solution

Create the temporary file **in the same directory as the target**, not in the system default temp location. Same directory guarantees same filesystem, which guarantees the rename is genuinely atomic rather than accidentally so.

For the secret-expansion issue, persistence was changed to write from the original, pre-expansion parse of the configuration — preserving variable references as references — and to refuse to write at all if a reference can't be represented that way, rather than silently downgrading it to a literal.

## 💡 Takeaway

- **Temp-file-then-rename is only atomic when both files are on the same filesystem.** Using the system default temp directory is a common, easy way to violate that silently — write the temp file next to its final destination instead.
- **A cross-filesystem rename doesn't degrade gracefully; it fails outright.** Treat that failure as expected on at least one popular platform, and make sure the failure path itself doesn't hold a resource open when it happens.
- **Anything that expands secrets for use must never accidentally become the thing that writes them to disk.** Persist from the pre-expansion representation, and prefer failing to write over silently downgrading a reference into a literal.
