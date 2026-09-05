---
title: "A Hard Invariant Encoded as a Score Is Only True Until Two Scores Tie"
collection: journal
permalink: /journal/partition-dont-rank-a-hard-invariant/
excerpt: "Certain items were supposed to always survive a selection pass, guaranteed. The rule that protected them checked their position after sorting by score — and two unrelated scoring factors summed to give the protected items and an ordinary item an identical value, so the guarantee held only by coincidence of the data."
---

**The trick:** if something must always be true — always kept, always first, always excluded — don't express that as a value competing in a ranked comparison. Partition it out before scoring, and only rank what's left.

## Issue

A selection pass over a list of candidates was supposed to guarantee that certain marked items always survived, regardless of how everything else scored. In production, that guarantee occasionally failed: a marked item was dropped, and an unmarked one kept in its place.

## Root Cause

The "always survives" rule was implemented as one more input to a composite score, alongside several other factors like recency and usage frequency — reasoning that giving it a very high fixed weight would push it to the top of the sort every time. Downstream, a separate check enforced the actual guarantee by looking at whether the protected items landed within the number of slots being kept — which only works if the sort reliably puts them there.

Two of the scoring factors, in combination, could sum to exactly the same total for a protected item and an ordinary one. Nothing in a stable or unstable sort resolves a tie in the direction the invariant needs; it resolves it however the sort implementation happens to order equal keys, which is an accident of algorithm and input order, not a decision anyone made. On the runs where the tie broke the wrong way, the protected item landed just outside the kept slots, and the position-based check waved it through as absent without complaint — because the check trusted the sort to have already handled the guarantee, when the sort was never actually guaranteeing it.

## Solution

Two-phase selection instead of one ranked pass: partition the candidate list into protected and unprotected **before** any scoring happens, keep every protected item unconditionally, and run the score-and-rank logic only over whatever slots remain for the unprotected ones. The guarantee no longer depends on where anything lands in a sort — it's structural, decided before the sort ever runs.

## 💡 Takeaway

- **Never encode a hard invariant as a value inside a comparison.** A score, however high, is still just a number competing with other numbers, and any two numbers can tie. A guarantee expressed that way is actually "usually true, and always true if nothing else happens to match this exact value" — which is not a guarantee.
- **Partition first, rank second, whenever some items are exempt from the ranking's outcome by rule rather than by merit.** It turns "we hope the sort puts these where we need them" into "these were never subject to the sort in the first place."
- **A downstream check that verifies an invariant by inspecting position after the fact is only as reliable as the mechanism that produced the position.** If that mechanism is a comparison that can tie, the check is verifying an accident, not a guarantee.
