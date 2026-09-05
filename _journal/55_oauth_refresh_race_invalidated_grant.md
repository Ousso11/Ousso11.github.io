---
title: "Two Refreshes Happened at Once. That Cost Us the Whole Login."
collection: journal
order: 3
permalink: /journal/rotating-credentials-need-single-flight/
excerpt: "Users were intermittently forced to re-authenticate from scratch, for no visible reason. A background refresh and a 401-triggered refresh could race each other, and the refresh endpoint rotates the token on every use — so the loser presented a credential the winner had already spent."
---

**The trick:** a rotating credential turns ordinary duplicate work into a destructive race. If the endpoint invalidates the old token on every rotation, the loser of a concurrent refresh doesn't waste a call — it burns the whole grant.

## Issue

Users were occasionally forced to fully re-authenticate, with no error explaining why and no correlation to anything they'd done.

## Root Cause

A background refresh ticker and a 401-triggered refresh could fire in the same window with nothing serializing them. The identity provider rotates the refresh token on every use, so whichever caller lost the race presented an already-spent token — indistinguishable from replay, invalidating the entire grant.

## Solution

Real single-flight: concurrent callers wait on one in-flight refresh instead of racing their own. The lock is never held across the network call. Before spending a token, re-read the credential store — a coexisting process may have already rotated it.

## 💡 Takeaway

- Any credential that rotates on use turns duplicate work into destruction. Single-flight is mandatory, not an optimization.
- Never hold a lock across a network call.
- When a credential might be shared across processes, re-check the store before spending your local copy.
