---
title: "${VAR:-default} Fires on Empty, Not Just Unset"
collection: journal
permalink: /journal/bash-default-fires-on-empty-not-just-unset/
excerpt: "A scope switch meant to skip a whole check silently never skipped it, because the variable it tested was set to an empty string rather than left unset — and bash's default-value syntax treats those as the same thing."
---

**The trick:** `${VAR:-default}` substitutes on empty *and* unset. If your script needs to tell the two apart, use `${VAR-default}` — no colon — which substitutes only when the variable is genuinely unset.

## Issue

A scope switch was supposed to let a pipeline stage skip a check entirely by leaving a variable unset. It never skipped. Worse, the file it pointed at didn't exist, and the stage reported a clean pass anyway — the lint had silently dropped out of the scan altogether while the gate kept reporting green.

## Root Cause

The condition guarding the check was written as `${TARGET_FILE:-}`, tested for emptiness, with the intent that an unset variable would leave it blank and skip the block. Bash's `${VAR:-default}` form has a colon, and the colon is what makes it check for **emptiness**, not just absence. An unset variable and a variable explicitly set to `""` are indistinguishable to that syntax — both substitute the default.

Somewhere upstream in the pipeline, the variable had been set to an empty string rather than left unset — a common, easy-to-miss distinction when a value is threaded through several layers of scripts and configuration. The empty-string case took the same branch as "properly configured," pointed the checker at a path that had already been deleted, and the check reported success on a target that was never actually checked.

A related trap sat one line away: `${ASSUME_YES:+--yes}` — the alternate-value form — treats **any non-empty string** as true, including the literal string `"0"`. A flag meant to require an explicit affirmative was satisfied by a value that reads, to a human, as a clear no.

## Solution

Switch to the colon-free form, `${VAR-default}`, wherever the distinction between "unset" and "explicitly set to empty" actually matters — which, for anything gating a security or correctness check, it usually does. For the `+` form, compare the value explicitly (`[[ "$ASSUME_YES" == "yes" ]]`) rather than relying on presence-of-any-string as a stand-in for a real boolean.

## 💡 Takeaway

- **`${VAR:-default}` and `${VAR-default}` are different operators, one character apart, and almost nobody remembers which is which under pressure.** The colon means "also treat empty as absent." Decide, deliberately, whether your variable's emptiness should mean the same thing as its absence — it often shouldn't, especially for anything that gates a check.
- **`${VAR:+alt}` treats every non-empty string as truthy, `"0"` and `"false"` included.** Never use it as a boolean test; compare the value explicitly against the string that means yes.
- **When a value travels through several layers of scripts before reaching the check that uses it, "unset" and "empty" tend to blur together on the way.** The bug isn't usually in the line that reads the variable — it's several files upstream, in whatever set it to `""` instead of leaving it alone.
