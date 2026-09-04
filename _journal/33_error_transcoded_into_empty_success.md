---
title: "A 400 That Arrived Looking Like an Empty 200"
collection: journal
permalink: /journal/format-transcoder-needs-an-error-arm/
excerpt: "When an upstream call failed, the client received a status 400 with a perfectly well-formed, completely empty streaming response — no error text anywhere. Only one of two response paths through the proxy had ever been taught what an error looks like."
---

## Issue

A specific request path through our proxy — one that reroutes a request and replays the result as a stream — turned upstream errors into something worse than an error. The client saw HTTP 400, paired with a well-formed streaming payload that opened and closed with no content in between. There was no error message to read anywhere in the response.

## Root Cause

The reroute path always pipes its response through a JSON-to-stream resynthesizer, regardless of whether the underlying call succeeded. The resynthesizer's job is to walk a JSON response's content blocks and turn them into the matching sequence of stream events.

An upstream error response has no content blocks — it's a completely different JSON shape, an error envelope rather than a message. The resynthesizer, written only against the success shape, looked for content it could not find and dutifully emitted a valid open-and-close pair of stream events around nothing. The actual error text was sitting right there in the response body; nothing had a reason to look at it.

The plain, non-rerouted path relayed errors correctly, which is exactly why this looked provider-specific and intermittent rather than like a proxy bug — only requests that happened to take the reroute path could ever hit it.

## Solution

Add an explicit check before the resynthesizer ever runs: any non-2xx status, or a body that matches the shape of an error envelope regardless of status, short-circuits straight to a dedicated error-event constructor. That constructor wraps the original error body verbatim in a single error frame and preserves the original status code untouched.

## 💡 Takeaway

- **A format transcoder needs an explicit error arm.** "Parse the success shape and emit whatever you found" silently degrades every unrecognized shape — most importantly, errors — into empty-looking success.
- **The absence of content is not the same as the absence of a problem.** A transcoder that only knows how to represent content will represent an error as *no* content, which is indistinguishable from a message that legitimately has nothing to say.
- **When two code paths handle the same logical operation differently, expect the untested one to be broken exactly where the tested one isn't.** The direct path worked because someone had reason to test errors on it; the reroute path existed for a different reason and inherited none of that scrutiny.
