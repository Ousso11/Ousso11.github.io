---
title: "An Unblinded LLM Judge Biased a Benchmark's Accuracy Comparison"
collection: journal
order: 8
permalink: /journal/blind-the-judge-and-measure-its-noise/
excerpt: "A benchmark's judge prompt labelled one answer as coming from the treated arm and printed the exact compression ratio applied to it. A judge primed to expect degradation will find it — and small differences between configurations turned out to be smaller than the judge's own noise floor."
---

**The trick:** blind an LLM judge completely — no arm labels, no treatment strength, randomized position — and measure the judge's own repeat-scoring noise before trusting any difference smaller than it.

## Issue

A benchmark comparing several treatment strengths produced small, consistent differences that drove real production recommendations — differences under a single point on a 0-to-1 scale.

## Root Cause

The judge prompt stated outright which answer came from the treated arm, and the exact treatment strength applied. A judge told "this was compressed 10x" is primed to find degradation. A pairwise variant always listed the same answer first with no randomization, letting any positional bias compound instead of cancel.

## Solution

Strip arm labels and treatment numbers from the judge prompt; randomize position across trials. Separately, measure the judge's own noise by repeat-scoring the same pair — and treat any difference smaller than that noise as not a finding.

## 💡 Takeaway

- Never tell a judge which arm it's grading or how strong the treatment was — both are answers to the question it's meant to answer independently.
- Randomize position in any pairwise LLM comparison; un-randomized order lets bias compound rather than cancel.
- Measure your judge's repeat-scoring noise before reporting a difference smaller than it.
