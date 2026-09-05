---
title: "The Unique Index Postgres Swore Didn't Exist"
collection: journal
permalink: /journal/partial-unique-index-and-on-conflict/
excerpt: "Every insert failed with 'no unique or exclusion constraint matching the ON CONFLICT specification' — against a column that demonstrably had a unique index on it. ON CONFLICT does arbiter inference, and a partial index only qualifies if the statement's own predicate implies the index's."
---

**The trick:** `ON CONFLICT` performs arbiter inference, not an index lookup. A partial unique index only works with an upsert that can restate its `WHERE` clause — most query builders and REST layers can't, so they need a total index instead.

## Issue

Right after shipping event-id deduplication for the billing pipeline, the background flush began quarantining every row. Postgres was returning:

```
42P10: no unique or exclusion constraint matching the ON CONFLICT specification
```

against a column that had a unique index on it. `\d usage_logs` showed the index. The insert named the same column. Postgres disagreed.

## Root Cause

The index had been created as a **partial** index:

```sql
CREATE UNIQUE INDEX ... ON usage_logs (event_id) WHERE event_id IS NOT NULL;
```

That is the natural choice — legacy rows predating the column have `NULL`, and a partial index seems like the polite way to exclude them.

But `ON CONFLICT` does not do an index lookup. It does **arbiter inference**: it will only use a partial index if the *statement's own predicate provably implies the index's predicate*. Our inserts went through a REST layer that emits a plain `INSERT ... ON CONFLICT (event_id) DO NOTHING` and has no way to attach a `WHERE event_id IS NOT NULL`. With no matching *total* index, inference fails and Postgres raises 42P10 rather than silently picking something close.

## Solution

Drop the partial index; recreate it without the `WHERE`.

This is safe because of a SQL-standard subtlety that made the predicate pointless to begin with: **NULLs are distinct in a unique index by default**. An unconditional `UNIQUE(event_id)` already permits unlimited legacy `NULL` rows. The `WHERE` bought nothing and cost us arbiter inference.

The inverse of this bug was living elsewhere in the same codebase, and is worth stating because it is the more dangerous direction. A payments table had a partial unique index `WHERE payment_id IS NOT NULL` used for idempotency. A checkout session that arrived with a `NULL` payment id therefore **escaped the constraint entirely** and could be credited twice. Same mechanism, and here the partial predicate did not produce an error — it produced a silent hole in an idempotency guarantee.

## 💡 Takeaway

- **`ON CONFLICT` infers an arbiter; it does not look up an index.** It must *prove* your statement's predicate implies the index's, and most ORMs and REST layers cannot express the predicate at all.
- **Before writing `WHERE col IS NOT NULL` on a unique index, remember the standard already treats NULLs as distinct.** The predicate is usually redundant and always costly.
- **A partial unique index used for idempotency has a hole exactly where the predicate excludes rows.** Every row outside the predicate is unconstrained — which is fine only if you have proved those rows cannot be duplicates.
- **A loud error is a gift.** The direction that raised 42P10 cost an afternoon; the direction that silently skipped the constraint cost real money.
