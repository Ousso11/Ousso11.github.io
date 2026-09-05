---
title: "A Bare lib/ in .gitignore Ate Sixteen Files"
collection: journal
permalink: /journal/gitignore-patterns-are-recursive/
excerpt: "A pipeline branch worked perfectly on the machine that wrote it and was unusable from a fresh clone. A gitignore entry of lib/ — added years earlier for build output — matches a directory of that name at any depth."
---

**The trick:** a `.gitignore` pattern with no leading slash matches at every depth in the tree. Anchor patterns to the root, and verify a clean clone against `git archive HEAD` — never your own working tree, which already has the files.

## Issue

A branch worked locally and was unusable from a clean clone. Every pipeline script failed immediately on a missing shared library file.

It surfaced by accident: a newly added CI job collected 311 tests where a local run collected 329.

## Root Cause

`.gitignore` contained a single line, added long before for build output:

```
lib/
```

A pattern with **no leading slash matches at every depth**. So it did not mean "the `lib` directory at the root of the repository". It meant "any directory named `lib`, anywhere". Sixteen files under `pipelines/lib/` were invisible to git — never staged, never committed, never noticed, because they existed on the machine of everyone who had created them.

Every pipeline script begins by sourcing a common file from that directory.

The archaeology was the interesting part. Someone had already hit this exact problem in a different directory and patched it with negations:

```
!security/lib/
!security/lib/**
```

That is a fix for one instance of the bug that leaves the cause intact — and being a blanket un-ignore, it also quietly committed three compiled bytecode files.

## Solution

**Anchor the pattern to the repository root**, which is what it always meant:

```
/lib/
/lib64/
```

The build output it was originally written for is already covered by another entry, so nothing was lost. Both negations were deleted.

Then three tests, written against the *class* of bug rather than this instance:

- every file under the pipelines directory is tracked by git
- every `source path/to/x.sh` in the tree resolves to a file git actually has
- no compiled bytecode is tracked

The second one is the valuable one, and the way it is verified matters: it exports `HEAD` to a scratch directory and resolves the paths **there**. Checking the working tree would have passed on the very machine where the bug was born.

## 💡 Takeaway

- **`.gitignore` patterns without a leading slash are recursive.** `lib/`, `build/`, `dist/`, `tmp/`, `node_modules/` — every one of them is a wildcard over your whole tree.
- **When you find yourself writing a negation to rescue a directory, the pattern above it is wrong.** The negation fixes today's symptom and leaves the trap armed for the next directory to fall into.
- **"Does a clean clone work" is a testable property, not a hope.** Test against `git archive HEAD`, never against the working tree — the working tree is exactly where the missing files still are.
- **Missing-file bugs are invisible to the author by construction.** Everyone who could reproduce it already had the files.
