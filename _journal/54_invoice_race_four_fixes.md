---
title: "Four Fixes to a Race That Kept Finding New Ways to Lose Money"
collection: journal
permalink: /journal/status-guard-is-not-a-version-guard/
excerpt: "Folding a revoked key's usage into an open invoice lost money under concurrency. Fixing the double-billing created silent under-billing; fixing that covered one of three race paths; and the last one was a plain lost update that a status guard cannot see."
---

**The trick:** guarding a write on *status* and guarding it on *version* solve two different problems — status stops illegal transitions, a version token stops lost increments. Money-handling code under concurrency usually needs both, and a fix that adds only one will look complete and still lose money.

## Issue

Revoking an enterprise API key folds that key's month-to-date usage into the customer's open monthly invoice. Under concurrency, money went missing — sometimes usage vanished, and sometimes an invoice that had already been **paid** was silently reopened and overwritten with new totals.

Four distinct bugs, found in sequence over about six hours. Each fix was correct and each one exposed the next.

## Root Cause

**1. Time-of-check to time-of-use.** The routine read the invoice, computed new totals in application code, and issued a blind `UPDATE ... WHERE id = ?`. A payment webhook settling that same invoice inside the window got clobbered: a `paid` invoice was rewritten to `pending` with new totals, and the customer was billed again for money already collected.

**2. The fix for (1) created a data-loss bug.** Adding a status predicate to the update made it atomic — the statement now correctly *refused* to touch a settled invoice. But the caller returned failure and dropped the usage on the floor. We had converted silent double-billing into silent under-billing. A conditional update that correctly refuses is only half a fix; you still owe an answer for the refused case.

**3. Recovery covered one of three race paths.** There was a terminal-status read path, a guarded-update path, and a create path. Only the guarded-update loser routed into recovery. A lost insert race and a terminal-status read still discarded the usage.

**4. A textbook lost update.** Two concurrent revokes on the same month both read a total of *X*, both computed *X + their own delta*, both passed the status guard, and the second overwrote the first. Status guards protect against illegal transitions, not against concurrent increments within the same state.

## Solution

One primitive, expressed entirely as predicates on a single statement so the check-and-set is atomic:

```
update_invoice(id, updates,
               allowed_statuses=[...],      # legal transition
               expected_updated_at=token)   # optimistic lock
```

The caller became a bounded retry that re-reads fresh totals on conflict, and the final losing attempt routes into recovery, which either folds the usage into a concurrently-waived invoice or bills the residual separately. Sub-cent residuals are skipped.

A small API consequence worth noting: the read had to be widened to return `updated_at`. The token you lock on has to come back from the read — optimistic locking is a property of the pair, not of the write alone.

Two follow-ons landed the same week, and they are the interesting part:

- A billing-exemption flag was captured once at the start; an administrator removing the exemption mid-flight still got the invoice waived. The flag is now re-evaluated at the top of every retry attempt.
- Removing an exemption deletes a row in a *different* table, which never touches the invoice — so `updated_at` on the invoice cannot possibly see it. No amount of row versioning helps. That one needed post-write verification and self-healing: after waiving, re-check the exemption and reinstate if it has gone, deriving the reinstated amount from a freshly re-fetched row so a concurrent fold isn't overwritten.

## 💡 Takeaway

- **Guarding on status and guarding on version solve different problems.** Status stops illegal transitions; a version token stops lost increments. Money code under concurrency usually needs both.
- **A refusal is not a resolution.** Making the write safe means the losing path now has a value in its hands that nobody has recorded. Decide where it goes before you ship the guard.
- **Optimistic locking only sees changes that touch the locked row.** When the invariant depends on state in another table, row versioning is structurally blind to it and you need post-write verification.
- **Re-read every input on retry, not just the one that conflicted.** Anything captured before the first attempt is a snapshot of a world that has since moved.
