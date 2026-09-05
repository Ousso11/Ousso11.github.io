---
title: "Two Refreshes Happened at Once. That Cost Us the Whole Login."
collection: journal
permalink: /journal/rotating-credentials-need-single-flight/
excerpt: "Users were intermittently forced to re-authenticate from scratch, for no visible reason. A background refresh and a 401-triggered refresh could race each other, and the refresh endpoint rotates the token on every use — so the loser presented a credential the winner had already spent."
---

**The trick:** a rotating credential turns ordinary duplicate work into a destructive race. If two callers can refresh the same token concurrently and the endpoint invalidates the old token on every rotation, the loser doesn't just waste an HTTP call — it burns the grant. Single-flight isn't an optimization here; it's the only correct behavior.

## Issue

Users were occasionally, unpredictably forced to re-authenticate from a fully logged-in state, with no error message that explained why. It didn't correlate with anything the user had done. Restarting sometimes fixed it, sometimes didn't.

## Root Cause

The token manager followed an ordinary-looking pattern: read the current credentials, call the refresh endpoint if they're near expiry, overwrite the stored credentials with the response. Nothing about that pattern accounts for two callers doing it at the same time — a background refresh ticker running on its usual schedule, and a separate refresh triggered immediately by a request that just got a 401. Both can fire within the same short window, and nothing serialized them.

That would just mean one wasted HTTP call almost anywhere else. Here it was destructive, because the identity provider **rotates the refresh token on every use** — a normal and correct security property for OAuth. Whichever caller lost the race presented a refresh token that the winner had already spent seconds earlier. From the provider's side, that's indistinguishable from token replay, and it invalidates the entire grant, not just the one request. The user is now logged out, with no path back except re-authenticating from scratch.

Three compounding issues made it worse once found. The lock protecting the credential state was held across the actual refresh HTTP call, so every reader blocked for the full round-trip even in the non-racing case. A failed refresh retried on the very next tick with no backoff, so a provider hiccup turned into a tight retry loop. And on one platform, credentials were read from the OS keychain but refreshed credentials were written only to a plaintext fallback file — so the keychain silently went stale while the file held the truth, and whichever path a given code path happened to read from determined whether it saw current or expired credentials.

## Solution

Real single-flight: concurrent callers requesting a refresh check whether one is already in progress and, if so, wait on that one's result instead of starting their own. The lock guarding shared state is never held across the network call — only around the bookkeeping before and after it. Failures back off exponentially rather than retrying on the next tick unconditionally. Credential writes go through both storage paths atomically.

The subtlest addition: **before spending a refresh token, re-read the credential store.** A separate process sharing the same credential file — another instance of the same tool, a coexisting CLI — may have already rotated the token out from under this process. Checking first turns "I might burn my last token on a token that's already dead" into "let me first see whether someone already handled this."

## 💡 Takeaway

- **Any credential that rotates on use converts a duplicate-work race into a destructive one.** Treat single-flight as mandatory the moment a token, nonce, or one-time secret is involved — not as a nice-to-have for reducing redundant calls.
- **Never hold a lock across a network call if you can help it.** It turns every concurrent reader into a queue behind your slowest dependency, for no benefit beyond enforcing exclusion you could have enforced more narrowly.
- **When two processes might share a rotating credential, re-check the shared source before spending it.** Someone else may have already done the work, and spending your local copy without checking wastes the one thing you can't get back.
- **Two storage locations for the same credential is two sources of truth waiting to disagree.** If a system reads from one store and writes to another, staleness isn't a bug in either path individually — it's a guaranteed property of the pair.
