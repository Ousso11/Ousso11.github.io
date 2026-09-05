---
title: "A Benchmark's n=200 Headline Was Really Two Different 200s"
collection: journal
permalink: /journal/ab-comparators-must-intersect-sample-ids/
excerpt: "A published cost-and-accuracy comparison had to be retracted and restated on a smaller n. A third-party search API ran out of quota mid-sweep, and the two arms being compared died at different points in their run — so neither the sample set nor the count of dropped questions matched."
---

**The trick:** before comparing two runs of anything, intersect their completed-item IDs. An equal sample count is not proof of an equal sample set.

## Issue

A published head-to-head comparison reported one configuration beating another on both accuracy and cost, at a stated sample size of 200 questions each. Days later it had to be retracted and restated at a smaller, different sample size, with a materially smaller accuracy gap.

## Root Cause

A third-party search API the benchmark depended on returned a payment-required error partway through the sweep, once the account's quota ran out. That failure happens at the search step, before either configuration's own processing begins, and the runner handled it the way it handles any per-question exception: record an error and move on, scored downstream as a wrong answer with no further API call.

The two configurations under comparison are not identical in how much work they do per question, so they consumed the shared quota at different rates and hit the cutoff at different points in the run. One arm lost a meaningfully different number of questions to the outage than the other. The result was two "n=200" result sets that were actually two different sample sets of two different sizes, compared as though they were the same 200 questions run under two configurations — which is the entire premise a head-to-head comparison depends on.

Nothing in the comparison tooling checked that both sides' result sets shared the same question identifiers, or even the same count, before printing a side-by-side summary. It printed whatever two dictionaries of aggregate statistics it was handed.

## Solution

Recompute both arms restricted to the intersection of questions where both configurations had actually completed a real run — the subset neither arm lost to the outage. Document explicitly that the dropped tail was unrelated to question content (the run order was shuffled), so restricting to the intersection doesn't introduce its own bias.

## 💡 Takeaway

- **Two runs reporting the same sample count are not automatically the same sample set.** Any infrastructure failure that can strike mid-run at different rates for different arms will silently substitute "a comparable-sized set" for "the same set."
- **A comparator that prints two summaries side by side should refuse to run until it has checked the two result sets actually agree on which questions they cover.** Compute the intersection yourself; don't trust that equal counts imply equal content.
- **An infrastructure failure scored as "the model got it wrong" is a silent corruption of your accuracy number, not a missing datapoint.** Distinguish "we asked and got a wrong answer" from "we never managed to ask" at the point of failure, not after the fact.
