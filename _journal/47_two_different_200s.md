---
title: "Two Runs of 200 Questions. Never the Same 200."
collection: journal
order: 10
permalink: /journal/ab-comparators-must-intersect-sample-ids/
excerpt: "A published cost-and-accuracy comparison had to be retracted and restated on a smaller n. A third-party search API ran out of quota mid-sweep, and the two arms being compared died at different points in their run — so neither the sample set nor the count of dropped questions matched."
---

**The trick:** before comparing two runs of anything, intersect their completed-item IDs. An equal sample count is not proof of an equal sample set.

## Issue

A published benchmark comparison reported one config beating another at n=200 on both cost and accuracy. Days later it had to be retracted and restated at a smaller, different n.

## Root Cause

A third-party search API ran out of quota mid-sweep. Both configurations hit that cutoff at different points, since they don't do equal amounts of work per question — so two "n=200" result sets were actually two different sample sets of two different sizes, compared as if identical. Nothing in the comparison tooling checked that both sides covered the same questions before printing a delta.

## Solution

Recompute both arms over the intersection of questions both actually completed, and confirm the dropped tail was content-independent so restricting to the intersection introduces no new bias.

## 💡 Takeaway

- Equal sample counts across two runs are not proof of an equal sample set.
- Any comparator printing a side-by-side delta should refuse to run until it's verified the two sides cover the same items.
- An infrastructure failure scored as "wrong answer" silently corrupts accuracy — distinguish "asked and got it wrong" from "never managed to ask."
