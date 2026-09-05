---
title: "Deposited $0. Balance $0. Spent $10.53."
collection: journal
order: 2
permalink: /journal/clamping-is-where-money-goes-to-die/
excerpt: "Wallets in production showed the impossible triple: deposited $0, balance $0, spent $10.53 — and kept serving. A clamp in the deduction routine absorbed every overshoot, and a write-behind cache turned the resulting lie into a renewable resource."
---

**The trick:** never let an operation that can't fully succeed report success. Return what was actually applied versus what was rejected — especially anywhere a clamp (`GREATEST`, `LEAST`, `min`, `max`) can silently absorb an overshoot.

## Issue

Production wallets showed an impossible state — zero deposited, zero balance, real spend on record — and kept serving requests indefinitely.

## Root Cause

The deduction routine clamped an over-withdrawal to zero with `GREATEST(0, balance - amount)`, charged the full amount to lifetime usage anyway, and returned success. The missing difference existed nowhere except as a mismatch between two columns nothing ever compared. A write-behind cache made it worse: every periodic resync pulled the clamped zero back from the database, handing out a fresh floor's worth of spending room each cycle.

## Solution

Rewrite the deduction to partially apply and report: return how much was actually applied and how much was rejected, rather than a boolean. A non-zero rejection now triggers an alert and evicts the resync cooldown that had been hiding the drift.

## 💡 Takeaway

- Clamping (`GREATEST`/`LEAST`) converts an invariant violation into a plausible value — the violation then only shows up as disagreement between two columns nothing checks.
- Return applied vs. rejected, never just success/fail, for anything that can partially fail.
- A cache periodically resynced from a store that can lie will keep renewing the lie.
