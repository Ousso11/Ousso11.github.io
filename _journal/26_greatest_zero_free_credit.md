---
title: "Deposited $0. Balance $0. Spent $10.53."
collection: journal
permalink: /journal/clamping-is-where-money-goes-to-die/
excerpt: "Wallets in production showed the impossible triple: deposited $0, balance $0, spent $10.53 — and kept serving. A clamp in the deduction routine absorbed every overshoot, and a write-behind cache turned the resulting lie into a renewable resource."
---

**The trick:** never let an operation that can't fully succeed report success. Return what was actually applied versus what was rejected — especially anywhere a clamp (`GREATEST`, `LEAST`, `min`, `max`) can silently absorb an overshoot.

## Issue

Production wallets were showing an impossible triple: **deposited $0, balance $0, spent $10.53** — and the accounts kept serving requests, indefinitely.

## Root Cause

The balance-deduction stored procedure did this:

```sql
balance_usd       = GREATEST(0, balance_usd - p_amount),
lifetime_used_usd = lifetime_used_usd + p_amount   -- unconditional, full amount
```

Two flaws that compound into a self-sustaining leak.

**The clamp silently absorbed the overshoot.** Deducting $5 from a $1 balance charged the full $5 to lifetime usage, removed only $1 from the balance, and **returned success**. The missing $4 existed nowhere in the system except as a disagreement between two columns that no code path ever compared.

**A write-behind cache then recycled the lie.** The hot balance lives in a cache; a background job flushes deltas to the database; and periodically the cache is resynced *from* the database. Because the database had clamped to zero, each resync handed the cache a fresh `0` — which, measured against a negative overdraft floor, is another full floor's worth of spending runway. **Every flush-and-resync cycle regenerated the user's credit line.** A short reload cooldown made the drift disappear again before anyone could catch it in the act.

## Solution

Rewrite the procedure to **partially apply and report**, rather than clamp and claim success:

```sql
v_room     := GREATEST(0, v_balance - p_floor);
v_applied  := LEAST(p_amount, v_room);
v_rejected := p_amount - v_applied;
-- lifetime usage increments by v_applied only
RETURNS TABLE(applied, rejected, new_balance, error_code)
```

Backwards compatible via a defaulted argument, so the old two-argument call still resolves during a log-only rollout.

On the application side, a non-zero `rejected` is now a first-class event: two counters, a critical log line, an audit row — and, crucially, it **evicts the reload cooldown entry** so the resync can no longer paper over the drift on the next cycle.

The success path costs nothing extra. All of the new work sits behind `rejected > 0`, which is empty in steady state. A separate batched migration repaired the ledgers of the accounts already affected.

## 💡 Takeaway

- **`GREATEST`, `LEAST` and `clamp` are where money goes to die.** Clamping converts an invariant violation into a plausible-looking value. The violation then exists only as disagreement between two columns, which nothing checks.
- **An operation that cannot fully succeed must return how much it actually did.** A boolean is not enough. `applied` and `rejected` are the difference between a leak and an alert.
- **When a cache is periodically resynced from a store that lies, the lie becomes renewable.** Look for loops where a derived value is written back to its own source.
- **Anti-drift measures hide drift.** The cooldown that smoothed the numbers was the reason nobody saw it for so long. Any mechanism that suppresses a discrepancy must be disabled the moment a real one is detected.
