---
title: "A 47-Second Reaction That Never Happened"
collection: journal
order: 26
permalink: /journal/lag-must-be-a-transition/
excerpt: "Two autoscaling configurations were credited with fast reactions to a traffic spike they had scaled up for minutes before the spike arrived. Fixing the measurement also meant admitting one number was 46s ±466s — useless precision the old output printed as a clean 47."
---

## Issue

After correcting *what* the autoscaling study measured, a second problem surfaced in *when* it measured.

The lag calculation scanned forward from the start of the measurement window looking for `desired > 1`. If a configuration had **already scaled before the window opened**, it matched on the very first sample — and was credited with a lightning-fast reaction it never made.

## Root Cause

This was not hypothetical. In a target-sizing sweep:

| Config | Reached `desired=2` | Reported reaction |
|---|---|---|
| Low target | **237 s before the spike** | 47 s |
| Mid target | **176 s before the spike** | 7 s |
| High target | after the spike | genuine |

Two of three configurations scaled up during the *plateau* — they were over-eager, provisioning capacity for traffic that hadn't arrived. Which is a real and interesting finding about target sizing. But the scorer reported it as **excellent responsiveness**, ranking the two most wasteful configurations highest on exactly the axis they were worst at.

A threshold crossing is not an event. Asking "when was the value above X?" answers a different question from "when did the value *become* above X?", and the difference only shows up when the initial state isn't what you assumed.

## Solution

**Lag now requires a transition** — above whatever was already desired at window start, not above a fixed constant.

**The eager case is now reported separately**, as its own field. This matters: it is a finding about the target, not a missing measurement. Suppressing it would have hidden the real result twice.

Then a third issue, found while validating the first two. **Capacity polling is irregular** — the gap between samples ranged from 20 s to 280 s within a single trial. A lag measured by polling is only accurate to the width of the gap it lands in. Attributing each lag to the polling gap it landed in produced re-derived numbers with honest error bars:

```
67 s  ± 20 s
209 s ± 81 s
46 s  ± 466 s   ← the old output printed this as "47 s"
```

That last row is the point of the whole exercise. The measurement is worthless — the polling gap it landed in is ten times the quantity being measured — and the previous version presented it as a clean two-digit number with no indication that it meant nothing.

## 💡 Takeaway

- **A threshold crossing is a transition, not a comparison.** Scanning for "value > X" silently matches a system that was already there.
- **Carry the error bar or you are making the number up.** Sampled measurements inherit the sampling interval; a lag of 46 s from a 466 s gap is not a lag, it is a rounding artifact wearing a number's clothes.
- **Report the anomaly rather than absorbing it.** "Scaled before the window" was a finding about target sizing. Folding it into the lag metric would have destroyed the finding *and* corrupted the metric.
