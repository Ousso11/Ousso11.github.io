---
title: "The Judge Already Knew the Answer It Was Grading"
collection: journal
permalink: /journal/blind-the-judge-and-measure-its-noise/
excerpt: "A benchmark's judge prompt labelled one answer as coming from the treated arm and printed the exact compression ratio applied to it. A judge primed to expect degradation will find it — and small differences between configurations turned out to be smaller than the judge's own noise floor."
---

**The trick:** blind an LLM judge completely — no arm labels, no treatment strength, randomized position — and measure the judge's own repeat-scoring noise before trusting any difference smaller than it.

## Issue

A benchmark comparing several configurations at different aggressiveness settings produced small but consistent differences that drove real recommendations about which setting to use in production. The differences were sub-single-point on a 0-to-1 scale.

## Root Cause

The prompt handed to the LLM acting as judge included, directly in its text, which of the two answers it was grading had come from the treated arm — and the exact strength of the treatment applied to it, printed as a number. A model told outright "this text was compressed 10 times over" is being handed the answer to the question it's supposed to be evaluating independently. It's primed to go looking for degradation, and a competent judge model will generally find something to justify what it's already been told to expect.

The pairwise variant of the same judge was worse: it stated which answer used the original input and which used the treated one, always in the same order, with no randomization and no swapping of positions across trials. Any positional bias the underlying judge model carries — and most carry some — applies uniformly across the entire comparison rather than washing out.

A separate issue compounded it on long documents. When the source text exceeded a length threshold, the judge's copy of the "original" reference was itself truncated down to a head-and-tail excerpt. The treatment being evaluated was specifically designed to preserve content from the middle of long documents — so on exactly the cases meant to showcase that behavior, the judge was being asked to verify facts against a reference that no longer contained them.

## Solution

The direct fix is to strip any label naming which answer belongs to which arm from the judge prompt, remove the explicit treatment-strength number, and randomize (or explicitly swap and average over) the position of each answer across judging trials so positional bias cancels out rather than compounds. Where the reference text has to be truncated for length, truncate it the same way the treatment does, so the judge isn't evaluating against a reference shaped differently from what it's judging.

Just as important: the judge's own noise was measured directly, by asking it to grade the identical pair of answers multiple times and looking at the spread of scores that produced. That measured noise floor became an explicit rule — differences between configurations smaller than the judge's own repeat-measurement variance are not reported as findings, because they aren't distinguishable from the judge simply being asked the same question twice.

## 💡 Takeaway

- **Never tell a judge which arm it's grading, or how strong the treatment was.** Both are answers to the question you're asking it to answer independently, and a judge that's told what to expect will generally find it.
- **Randomize or swap position in any pairwise comparison.** Positional bias in an LLM judge is common enough to assume present until proven otherwise, and an un-randomized order lets it apply in the same direction on every single trial.
- **If you truncate the reference for length, apply the same truncation logic the treatment itself uses.** Otherwise you risk judging the treatment's key behavior against a reference that specifically doesn't showcase it.
- **Measure your judge's own noise before reporting a difference smaller than it.** A repeat-measurement spread is cheap to collect and is the only honest way to know whether a headline number is signal.
