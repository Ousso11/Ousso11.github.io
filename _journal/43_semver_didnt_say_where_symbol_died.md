---
title: "The First Version Pin Was Wrong Because Semver Doesn't Say Where a Symbol Died"
collection: journal
permalink: /journal/bisect-the-pin-dont-guess-from-semver/
excerpt: "A CI job broke on an ImportError two dependency layers removed from any code we owned. The first pin capped the minor version, assuming that's where a removed symbol had landed. It hadn't. The removal shipped in a patch release."
---

## Issue

A CI job failed at module import time with a symbol no longer found in a third-party library. Nothing in our own code referenced that symbol directly — it broke two layers down, through an optional dependency of an optional dependency.

## Root Cause

A framework we depend on transitively had removed an internal utility function. One of our own optional integrations pulls in a proxy library that imports that function at its own module load time, so the whole integration became unimportable the moment that framework updated.

The first attempted fix capped the framework's version below the next minor release, on the reasonable-sounding assumption that a breaking removal like this would land at a minor boundary. CI kept failing. The cap had resolved to a version that, on inspection, had already dropped the symbol — the removal had shipped in a **patch** release, not a minor one. Semantic versioning tells you what a maintainer promises about compatibility; it does not tell you which specific release actually made a specific change, and an internal utility function is exactly the kind of thing that isn't covered by any compatibility promise at all.

A related, earlier issue in the same dependency chain: the optional integration originally listed the base library as its requirement rather than the specific installable variant that includes the module doing the importing. Anyone installing the plain extra got a `ModuleNotFoundError` for a completely unrelated third-party package on first use — the code path that needed it only existed behind a different extra than the one being pulled in.

## Solution

Bisect against the actual releases rather than reasoning from version semantics: install each candidate version in isolation and check directly whether the symbol is present. The correct cap was several patch releases earlier than the first guess. Pinned explicitly, with a comment naming the removed symbol and why the bound exists, so the next person to consider relaxing it knows what to check first.

For the missing-extra issue, the dependency declaration was corrected to name the specific installable variant that guarantees the code path actually needed is present.

## 💡 Takeaway

- **Semver tells you about promised compatibility, not about where a specific symbol died.** Internal utilities, private modules, and anything not covered by the public API contract can disappear in a patch release with no warning.
- **Bisect against real releases when a pin needs to be precise.** Install candidates and check directly; don't infer a boundary from version-number conventions.
- **When a library's optional extras gate which internal modules exist, name the specific extra your code actually needs.** "The base package" and "the package plus the extra that makes your import path work" are different dependencies, and declaring the wrong one only fails for users who exercise the feature.
