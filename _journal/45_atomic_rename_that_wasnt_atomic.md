---
title: "[19] Cross-Filesystem Temp Files Broke an 'Atomic' Config Write on Linux"
collection: journal
order: 19
permalink: /journal/temp-file-rename-across-filesystems/
excerpt: "A config persistence routine used the standard temp-file-then-rename pattern for an atomic write. On Linux, where the system temp directory is routinely a different filesystem from the target, the rename fails outright — and the failure path was the one that caused the deadlock in an earlier story."
---

**The trick:** a rename is only atomic within one filesystem. Write your temp file into the same directory as the final target, never into the system default temp directory.

## Issue

A config-persistence routine used the standard temp-file-then-rename pattern for an atomic write. On Linux specifically, it reliably failed.

## Root Cause

The temp file was created in the system's default temp directory, commonly its own filesystem — often an in-memory one. Renaming across filesystems isn't atomic; it isn't even a rename. It fails outright with a cross-device error, since a rename can only relink within one filesystem.

## Solution

Create the temp file in the *same directory* as the target. Same directory guarantees same filesystem, which guarantees the rename is genuinely atomic.

## 💡 Takeaway

- Temp-file-then-rename is only atomic when both files share a filesystem — the system temp directory is a common way to violate that silently.
- A cross-filesystem rename fails outright rather than degrading gracefully; expect it on Linux specifically.
- Check whether the failure path of an "atomic" operation can itself hold a lock open.
