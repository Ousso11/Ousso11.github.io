---
title: "The Local Stress Harness Was Lying About Failure Rates"
collection: journal
permalink: /journal/bound-cumulative-backoff-not-each-sleep/
excerpt: "On-premise customers saw 40-60% request failure during bursts. The local stress harness showed none of it, because it never generated enough concurrent load to trigger the exact queueing signal that real traffic hit constantly."
---

**The trick:** bound the cumulative backoff across every retry attempt, not just each individual sleep. A server-supplied wait hint is attacker- or bug-controlled input, and it needs a hard ceiling.

## Issue

Deployments handling real burst traffic saw 40 to 60 percent of requests fail outright. A local load-testing harness, run before every release, showed nothing resembling this. The two environments disagreed completely about whether there was a problem.

## Root Cause

The transport treated a specific "temporarily overloaded" response as an outright failure, raised immediately on the first occurrence. That response is not a failure in the traditional sense — it's a queueing signal, returned deliberately once a deployment's in-flight request cap is reached, meaning "you are asking faster than I can currently serve; wait and retry."

The local harness never generated enough concurrent load to hit that cap even once. It exercised the API correctly and thoroughly, and it simply never reproduced the one condition — sustained concurrency above a threshold — that real burst traffic hit constantly. A test environment that cannot reach a system's actual limits will never see what happens past them.

## Solution

Add real retry behavior: bounded exponential backoff with jitter, retrying specifically on the overload signal and on rate-limiting, honoring a server-provided wait hint from either the header or the response body, and clamping that hint to a maximum so a misbehaving server cannot pin the client into a multi-minute sleep by requesting an unreasonable delay.

A second act followed. The per-attempt cap wasn't sufficient on its own — a server returning a maximal wait hint on every single attempt could still stack several long sleeps back to back, one per retry, and the cumulative wait ballooned even though each individual sleep was bounded. The fix tracks total time already spent sleeping across all attempts and stops retrying once the cumulative figure would exceed the overall timeout, rather than only bounding each sleep in isolation.

## 💡 Takeaway

- **A local test environment that can't reach production's actual concurrency will never exercise production's actual failure modes.** If a defense mechanism exists for load your test never generates, your test cannot validate that defense.
- **A server-provided wait hint is attacker- or bug-controlled input, not a trusted instruction.** Cap it, always — retry behavior driven by an unbounded external value is one misbehaving dependency away from a client that never returns.
- **Bound the cumulative backoff across all attempts, not just each individual sleep.** A per-attempt cap that's respected every time can still add up to an unacceptable total wait if nothing tracks the running sum.
