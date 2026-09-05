---
title: "[7] A Missing Version Guard Let Concurrent Invoice Updates Lose Money"
collection: journal
order: 7
permalink: /journal/status-guard-is-not-a-version-guard/
excerpt: "Folding a revoked key's usage into an open invoice lost money under concurrency. Fixing the double-billing created silent under-billing; fixing that covered one of three race paths; and the last one was a plain lost update that a status guard cannot see."
---

**The trick:** guarding a write on *status* and guarding it on *version* solve two different problems — status stops illegal transitions, a version token stops lost increments. Money code under concurrency usually needs both.

## Issue

Revoking an API key folds its usage into an open invoice. Under concurrency, money went missing — sometimes usage vanished, sometimes an already-**paid** invoice was silently reopened and overwritten.

## Root Cause

Four bugs, found in sequence over six hours. A blind update let a payment webhook race the revoke and get overwritten (double-billing). Guarding the update on invoice status fixed that but dropped the usage on the floor when refused (under-billing). Recovery only covered one of three race paths. And a plain lost update remained: two concurrent revokes both read the same total, both passed the status guard, and the second silently overwrote the first.

## Solution

One atomic update with both a status guard and an optimistic-lock version token, retried with fresh reads on conflict, with the loser routed to recovery rather than discarded.

## 💡 Takeaway

- Status guards stop illegal transitions; version tokens stop lost increments. Concurrent money code usually needs both.
- A refusal is not a resolution — decide where the refused value goes before shipping the guard.
- Optimistic locking is blind to invariants that live in a different table; those need post-write verification instead.
