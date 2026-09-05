---
title: "The Enterprise Upgrade That Demoted Our Biggest Customers"
collection: journal
order: 38
permalink: /journal/upgrade-that-demoted/
excerpt: "Approving an enterprise key wrote tier3 unconditionally. For a customer already on tier4 or tier5, the upgrade was a downgrade — a rate-limit cut delivered by the function whose name promised the opposite."
---

## Issue

The helper invoked as a side effect when an enterprise key is approved wrote **`tier = tier3` unconditionally**.

For most accounts that is the intended effect: enterprise approval raises you to the enterprise floor. For an account already at **tier4 or tier5**, it was a **silent demotion** to that floor.

The function did what its name says for the common case, and the exact opposite for the customers who matter most. The failure surfaces later and elsewhere — as unexplained rate limiting on the largest accounts, with nothing in their history to explain it except an event that was logged as an *upgrade*.

## Root Cause

The word "floor" is doing a lot of work in the requirement, and the code dropped it.

"Enterprise customers get at least tier3" was implemented as **"enterprise customers get tier3"** — an assignment where a **monotone raise** was meant. The two agree on every input below the floor, which is the entire set of accounts anyone tested with. Tier4 and tier5 accounts are rare, valuable, and disproportionately unlikely to appear in a fixture.

There is a general shape here: *"ensure at least X"* implemented as *"set to X"* is correct on the majority of inputs and wrong precisely on the ones you can least afford to be wrong about.

## Solution

The **automatic side effect** now takes the higher of (current tier, enterprise floor), so it raises to the enterprise floor when needed and **no-ops when the current tier is already at or above tier3**.

**Admin-driven paths are deliberately unchanged.** An explicit operator override can still move a tier in either direction — an operator lowering a tier is an intentional act; an automatic side effect doing it is a bug. Making the automatic path monotone while leaving the explicit path bidirectional is the distinction that actually encodes the requirement.

Test coverage is the full matrix rather than the happy path:

| Starting tier | Expected |
|---|---|
| tier1 / tier2 | → tier3 |
| tier3 | no-op |
| **tier4 / tier5** | **stays put** |
| missing row | defaults to tier1 |

## 💡 Takeaway

- **"At least X" is not "= X".** Any automatic privilege adjustment should be monotone in the direction it advertises.
- **Enumerate the states, not the story.** A four-line test matrix catches this instantly; a test of the intended user journey never will.
- **Automatic and explicit paths deserve different rules.** Bidirectional is right for an admin action and wrong for a side effect.
- **The rarest states are the expensive ones.** tier4/tier5 accounts were the least represented in tests and the most costly to get wrong.
