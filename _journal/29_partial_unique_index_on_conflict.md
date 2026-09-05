---
title: "The Unique Index Postgres Swore Didn't Exist"
collection: journal
permalink: /journal/partial-unique-index-and-on-conflict/
excerpt: "Every insert failed with 'no unique or exclusion constraint matching the ON CONFLICT specification' — against a column that demonstrably had a unique index on it. ON CONFLICT does arbiter inference, and a partial index only qualifies if the statement's own predicate implies the index's."
---

**The trick:** `ON CONFLICT` performs arbiter inference, not an index lookup. A partial unique index only works with an upsert that can restate its `WHERE` clause — most query builders and REST layers can't, so they need a total index instead.

## Issue

Every insert into a billing pipeline started failing with `42P10: no unique or exclusion constraint matching the ON CONFLICT specification` — against a column that visibly had a unique index.

## Root Cause

The index was partial: `UNIQUE (event_id) WHERE event_id IS NOT NULL`. `ON CONFLICT` doesn't do a lookup — it proves the statement's predicate implies the index's. Our inserts went through a REST layer that emits a bare `ON CONFLICT (event_id)` with no `WHERE`, so inference failed outright.

## Solution

Drop the partial predicate. It bought nothing: NULLs are already distinct in a plain unique index, so an unconditional `UNIQUE(event_id)` already tolerates unlimited legacy NULL rows.

## 💡 Takeaway

- `ON CONFLICT` needs arbiter inference, which most REST/ORM layers can't express against a partial index.
- Before writing `WHERE col IS NOT NULL` on a unique index, remember NULLs are already distinct by default.
- The inverse failure is worse and silent: a partial unique index used for idempotency lets rows outside its predicate double-insert with no error at all.
