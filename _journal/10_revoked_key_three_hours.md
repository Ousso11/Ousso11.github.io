---
title: "A Revoked Key That Kept Working for Three Hours"
collection: journal
permalink: /journal/revoked-key-cache-window/
excerpt: "On-premise deployments cached a successful key validation for three hours, so revoking a leaked key did nothing for three hours. The same TTL was also the offline tolerance window — so lowering it would have broken the product's core promise."
---

## Issue

The on-premise proxy caches a successful API-key validation for a configurable TTL, and has no other channel through which to learn about revocation — that is what "on-premise" means.

That TTL was **three hours**.

Revocation almost always means **the key leaked**. Three hours of continued access after you have decided a credential is compromised is not an acceptable window.

## Root Cause

The interesting part is why it had not simply been lowered already.

**The same TTL served two unrelated purposes.**

1. **Freshness** — how stale a validation result may be before we re-check.
2. **Outage tolerance** — how long we keep serving when the control plane is unreachable.

On-premise customers run in restricted networks precisely so that their availability does not depend on someone else's control plane. If the platform is unreachable for twenty minutes, compression must keep working; that is the product.

So lowering the TTL to fix revocation would have **capped outage tolerance at the same small number** — turning a security fix into an availability regression, and breaking the single property on-premise exists to provide. One variable, two requirements pulling in opposite directions, and any single value is wrong for one of them.

## Solution

**Separate the two windows**, because they were never the same thing:

- **Re-validate every 15 minutes** — the freshness window.
- **Keep serving from cache for up to 3 hours** when the platform is unreachable — the fail-open window.

In the healthy case the proxy re-checks four times an hour and learns about revocation quickly. In an outage it keeps serving on the last known-good answer for exactly as long as it did before.

**Revocation exposure drops 12×**, at a cost of **4 validation calls per key per hour** — a rounding error against compression traffic.

Both windows are covered by tests, because the failure mode of this fix is subtle: it is easy to write a cache that re-validates on schedule but *also* stops serving when re-validation fails, which silently reintroduces the availability problem the separation was meant to preserve.

## 💡 Takeaway

- **When one constant serves two requirements, it is wrong for at least one of them.** The three-hour TTL was defensible as outage tolerance and indefensible as a security window; it was the same number.
- **Look for the reason a known problem wasn't fixed.** "Three hours is too long" was obvious. The coupling to availability was the actual bug.
- **Security fixes have availability side effects.** The version of this that just lowered the TTL would have passed review and broken customers.
