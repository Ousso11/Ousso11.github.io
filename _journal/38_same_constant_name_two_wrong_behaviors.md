---
title: "Same Constant Name, Two Different Wrong Behaviours, One Per Language"
collection: journal
permalink: /journal/cross-language-parity-hides-in-constant-names/
excerpt: "A Python and a TypeScript SDK each carried a STREAM_TIMEOUT constant. In Python it was pure dead code nobody's transport referenced. In TypeScript it was wired in, and it silently overrode every user-supplied timeout on the one code path where the name matched."
---

## Issue

Users setting a custom timeout on the SDK found it respected on ordinary calls and ignored on streaming calls — but only in one of the two language implementations. In the other, a configuration knob existed, looked correct, and did nothing at all.

## Root Cause

Both SDKs, maintained in parallel, had independently grown a constant with an identical name meant to bound how long a streaming request could run.

In the Python implementation, the constant was declared and never referenced by any transport code. It was dead — a config value with no consumer, which made it look like streaming had its own deliberate timeout behavior when in fact streaming used no timeout logic distinct from anything else.

In the TypeScript implementation, the same-named constant was very much alive: the streaming client used it directly to schedule an abort, hard-coded, while the ordinary request path used the user's configured timeout value. Two request paths, two different timeout sources, one of them fixed and invisible to configuration.

The parity failure is the interesting part. Neither implementation was obviously broken on its own — Python's was inert, TypeScript's was functioning exactly as written. It took comparing the two side by side to see that the same concept had been implemented in two incompatible ways, hiding behind an identical name that suggested they agreed.

## Solution

Delete the constant in both languages. Streaming now uses the same per-client timeout configuration as every other request path, in both SDKs. The regression test asserts on the actual scheduled delay value passed to the timer primitive, checking that it reflects a client-level override rather than any hard-coded default — a cheap, direct way to test a timeout without waiting for one to fire.

## 💡 Takeaway

- **Cross-language parity bugs hide in shared naming, not shared code.** Two SDKs with the same public API but independent implementations will drift, and an identical constant name across them is not evidence of identical behavior — it may be evidence of two different bugs that happen to rhyme.
- **Dead configuration is as dangerous as wrong configuration.** A constant nothing reads creates the impression of a deliberate design decision where none exists.
- **Test a timeout by asserting on the scheduled delay, not by waiting for the timeout to occur.** Spying on the underlying timer call is fast, deterministic, and catches exactly this class of bug — a hard-coded value silently substituted for the configured one.
