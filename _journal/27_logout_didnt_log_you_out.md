---
title: "Logging Out Didn't Log You Out, Twice"
collection: journal
permalink: /journal/logout-enum-typo-and-wrong-cache-tier/
excerpt: "The logout endpoint returned 200 and the API key kept working. First cause: an enum member that does not exist, raising inside a try whose except returned a success message. Second cause, thirteen minutes later: the invalidation cleared a different cache tier from the one the hot path reads."
---

## Issue

A user logs out. The endpoint returns `200 {"message": "Logout processed"}`. The API key keeps working.

## Root Cause

**Bug one: an enum member that does not exist.**

The handler set the key's status to `APIKeyStatus.REVOKED`. There is no `REVOKED` member — the enum has `PENDING`, `ACTIVE`, `EXPIRED` and `DECLINED`. So the line raised `AttributeError` at runtime, inside a `try` whose `except` returned a friendly `"Logout processed; key state unchanged."` with HTTP 200.

Two ordinary mistakes producing a security control that fails open *and* fails silent: an enum typo invisible to any linter that does not resolve members, laundered into a success response by an over-broad exception handler.

The fix was to stamp `revoked_at`, matching how the existing revoke path already worked and matching the `revoked_at IS NULL` filter every lookup already applied.

**Bug two, thirteen minutes later: the right invalidation, aimed at the wrong namespace.**

The database write now worked, and the key *still* kept working for the length of a cache TTL. Authentication is cached in tiers. The logout path invalidated `api_key_auth:{hash_of_key}`. The hot path in the validator reads a different tier: `api_key_validation:{token_id}`.

The invalidation call was real, it succeeded, and it cleared a cache nobody was reading on that path. This is the worst shape a cache bug can take — the fix looks present in the diff and does nothing.

The fix was to call the same helper the existing revoke flow used, which clears the validation, daily-limit and auth tiers together.

A sibling of the same bug lived nearby: naturally expiring keys kept working for about fifteen minutes, because the cached authentication payload did not carry `expires_at` at all. On a cache hit there was nothing to re-check.

## Solution

- Write `revoked_at`, and delete the code path that could have written a non-existent status.
- Narrow the exception handler so a failure to revoke is a 500, not a 200. **A security endpoint must not have a success-shaped fallback.**
- Route every invalidation through **one helper that clears all tiers**, and forbid open-coding cache keys at call sites.
- Include the fields the hot path needs to re-check — expiry among them — in the cached payload, so a hit can still say no.

## 💡 Takeaway

- **A `try/except` that returns a success message on failure is worse than a crash.** It makes the control fail open and removes the evidence in one move.
- **Cache invalidation is a correctness API, not a performance one.** When tiers are keyed differently — by key hash here, by token id there — invalidation belongs behind a single helper that every write path calls.
- **A cached authorisation decision must carry everything needed to re-evaluate it.** Otherwise a cache hit is a decision nobody is allowed to revisit.
- **Verify security fixes from the outside.** Both bugs presented identically to the user: a 200 and a key that still worked. Only an end-to-end "revoke, then call the API" check distinguishes them.
