---
title: "[23] Last Year Was Smaller Than Last 6 Months"
collection: journal
order: 23
permalink: /journal/postgrest-silent-row-cap/
excerpt: "A usage dashboard reported less traffic for a longer period than a shorter one. The cause was a silent 1000-row cap in PostgREST combined with ASC ordering, which dropped exactly the newest days — and it was one query in a class of fifteen."
---

## Issue

A user's usage page showed **less total usage for "last year" than for "last 6 months."** The longer window is a strict superset of the shorter one, so this is not a rounding disagreement or a timezone edge — it is arithmetically impossible.

Impossible results are the good kind of bug. They cannot be explained away, and they point at a mechanism rather than a judgement call.

## Root Cause

The daily-rollup query read its stats table **unpaginated**. PostgREST applies a **silent default cap of 1000 rows** — no error, no truncation flag, just fewer rows than you asked for. The cap is a server-side `max-rows` setting, so it is invisible from the client: the response is a well-formed 200 with a complete-looking array.

The rows came back in **ASC order**. So a query spanning 365 days returned the *oldest* 1000 rows and dropped the newest ones. Six months fit under the cap and was complete; a year did not, and lost precisely the recent, high-traffic days that a growing account contributes most of its usage in.

Two independently reasonable decisions — a server-side row cap and ascending order — combine into a result that is silently wrong in the direction least likely to be questioned.

The important part came next: **this was not one query.** The same unpaginated read pattern appeared in 15+ call sites, including invoice money paths. A dashboard that undercounts is embarrassing. An invoice that undercounts is a refund.

## Solution

Fix the class, not the instance.

**A single paginating helper** in the shared database base module, returning a frozen result object carrying `rows`, `truncated` and `count` — so truncation becomes a value the caller must look at rather than an absence they can't see. Two modes:

- **offset mode** for stable tables;
- **keyset cursor mode** for append-heavy log tables, which are written to *while you are draining them*. Offset pagination against a live-append table both skips and duplicates rows: every insert below your current offset shifts the window, so page N+1 re-reads rows you already have and misses others entirely. A keyset cursor pages on `WHERE (ts, id) > (last_ts, last_id)` instead, which is stable under concurrent inserts.

**Money paths fail closed.** Invoice generation now refuses to persist a partial sum on truncation rather than quietly billing the visible subset.

**Ordering made deterministic.** A migration adds `ORDER BY` tie-breakers on the time-series RPCs. This is subtle but load-bearing: Postgres makes no ordering guarantee between rows with equal sort keys, so two pages of a paginated read can independently order the same tied rows differently — dropping some and repeating others even though every page was fetched successfully.

**Time-series drains re-verify.** At end of drain, page 0 is re-read once and retried, to catch rows that landed mid-drain.

And while in there, the underlying reason the bug was hard to reason about at all: **three divergent period resolvers with three disagreeing day-count tables.** "Last 6 months" meant a different number of days depending on which code path you entered through. These collapsed into one backend resolver with a single table and timezone-aware semantics, plus one matching frontend module so the two cannot drift. An unknown period now returns **422 instead of silently collapsing to 1 day**, and an off-by-one (`days-1`) was removed.

## Verification

Backend 2,959 tests passing. New suites specifically for the failure modes:

- a **superset property** test — a longer period must never report less than a shorter one it contains;
- keyset pagination under **mid-drain inserts**;
- invoice **fail-closed** on truncation;
- a **>1000-row** consistency test, sized deliberately past the cap;
- non-UTC rendering.

## 💡 Takeaway

- **Silent caps are the most dangerous default in data access.** A limit that returns success with fewer rows will be discovered by an accountant, not a test.
- **When you find one, grep for the pattern, not the symptom.** The reported bug was a dashboard. The expensive instance was an invoice.
- **Order interacts with truncation.** `ASC` + a cap silently discards the newest data — usually the data that matters most.
- **Impossible outputs are a gift.** "Longer period, smaller number" is a property you can assert in a test forever.
