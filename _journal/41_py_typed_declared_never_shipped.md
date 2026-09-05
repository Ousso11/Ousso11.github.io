---
title: "A Type Marker Declared for a Year, Never Actually Shipped"
collection: journal
permalink: /journal/package-data-globs-fail-open/
excerpt: "Consumers running a type checker against a fully-annotated SDK got 'missing library stubs or py.typed marker'. The manifest had declared the marker file since the very first release. The file itself did not exist, and the packaging tool never once complained."
---

**The trick:** verify what you actually shipped by inspecting the built artifact, never by re-reading the manifest that was supposed to produce it. A package-data glob that matches nothing never warns you.

## Issue

Consumers running a type checker against our SDK saw the standard warning for an untyped package: missing stubs, missing marker file. The package was, in fact, fully annotated throughout. Something about how it was built was telling every downstream type checker otherwise.

## Root Cause

The packaging manifest had declared, since the very first release, that a specific marker file should be bundled into every distribution — the file that tells a type checker "trust the annotations in this package." The declaration had been correct and present for a year across multiple releases.

The file it named did not exist anywhere in the source tree.

The packaging tool's manifest declarations are glob patterns matched against what's on disk at build time. A pattern that matches nothing simply contributes nothing to the built package — silently. There is no warning for "this manifest entry never matched a single file," in any release, ever. The declaration had been a complete no-op since the day it was written, and nothing about the release process was positioned to catch it, because everyone was checking the manifest, and the manifest was correct.

## Solution

Create the file. It's empty by convention — its only job is to exist.

The lasting fix is the release-verification habit that came out of it: check the built artifact, not the manifest that describes it. `unzip -l` on the actual distribution file, checking for the marker's presence, catches exactly this class of bug. A downstream type-checker smoke test against the installed package is the more thorough version of the same idea.

## 💡 Takeaway

- **Manifest declarations that match files by pattern fail open.** A glob that matches nothing is indistinguishable, at build time, from a glob that correctly matched an empty file. Nothing warns you.
- **Trust the artifact, not its description.** The fastest way to know what you actually shipped is to open the file you shipped, not to re-read the configuration that was supposed to produce it.
- **A release gate should assert on outputs, never on intentions.** "The manifest says X is included" and "X is in the archive" are different claims, and only the second one is true when a user runs `pip install`.
