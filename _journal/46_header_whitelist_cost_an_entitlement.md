---
title: "A Header Whitelist Proxy Quietly Cost Users Their Entitlement"
collection: journal
permalink: /journal/whitelist-proxy-is-a-semantic-filter/
excerpt: "Requests that worked fine sent directly returned a paywall error through our proxy — but only on larger sessions. The proxy forwarded an explicit whitelist of headers rather than everything inbound, and several it dropped were exactly what the upstream API used to recognise a legitimate first-party client."
---

## Issue

Requests that worked perfectly when sent directly to the upstream API returned a usage-limit error when routed through our proxy — and only on larger sessions that had opted into an extended-context feature. The request bodies were identical either way.

## Root Cause

The proxy forwards headers through an explicit allowlist rather than passing the full inbound set through untouched. That's a defensible default for a proxy sitting between a client and a third-party API — you generally don't want to blindly relay everything a client sends.

Several headers missing from the list turned out not to be incidental telemetry. The upstream API uses a client's user-agent string, an application identifier, a session identifier, and a family of SDK-fingerprint headers to recognize that a request is coming from a legitimate first-party client application — and that recognition is what grants certain requests a subscription-based entitlement rather than metering them against a pay-per-use limit. Strip those headers, and the exact same request looks like it came from a generic API client instead, and gets denied the entitlement it should have had. The failure surfaced specifically on the extended-context feature because that feature is opt-in via its own header, and a request carrying that header without the identifying ones alongside it was the combination that tripped the limit.

## Solution

Add the missing identification headers to the forwarded list. Verified directly: the same request that previously came back denied returned success once those headers passed through unmodified.

## 💡 Takeaway

- **A whitelist proxy is a semantic filter, not a transparent one.** Every header you choose not to forward is a header you're betting doesn't matter to the other side — and headers that look like pure telemetry can be load-bearing for authorization or entitlement.
- **When a request behaves differently through a proxy than sent directly, diff the complete header set first**, before looking anywhere else. It's the cheapest possible check and it's exactly where this bug was.
- **Recognition-based entitlements are invisible in documentation and easy to break by accident.** If an upstream API grants anything based on who it thinks is calling, a proxy sitting in the middle needs to actively preserve whatever signals establish that identity.
