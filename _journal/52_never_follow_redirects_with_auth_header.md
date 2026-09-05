---
title: "A Trailing-Slash Redirect Can Re-Send Your API Key to a Different Host"
collection: journal
permalink: /journal/never-follow-redirects-with-an-auth-header/
excerpt: "An HTTP client SDK followed redirects transparently, the way most clients do by default. Every one of those redirects carried the caller's API key with it — including redirects the server never intended as a trust boundary, like a routine trailing-slash normalization."
---

**The trick:** any client SDK that attaches a credential header should refuse to follow redirects, full stop. A 3xx response should surface as a result to handle, never as an invisible second request to a URL the caller never approved.

## Issue

A routine review of an HTTP client library's transport layer turned up a quiet, structural exposure: every authenticated request was configured to follow redirects automatically, which is the default behavior of most HTTP client libraries and is exactly the kind of setting nobody thinks to question.

## Root Cause

An HTTP client following a redirect resends the original request — including its headers — to whatever URL the server names in the response, by default. That's convenient and mostly harmless for a browser navigating between pages. It's a different proposition for a client attaching a static, long-lived API key to every request: the redirect target is chosen entirely by the far end, and the client has, by default, no opinion about whether that target deserves the same credential the original request did.

The specific trigger that surfaces this is mundane. A trailing-slash mismatch, a domain migration, a load balancer's routing change — any of the ordinary reasons a server issues a 3xx — and the credential silently follows along to wherever that response points, cross-host if the response says so. None of these are attacks; they're routine server behavior that an unquestioned default turns into a credential-forwarding mechanism.

## Solution

Disable automatic redirect-following on every client that attaches an authentication header, and treat any 3xx response as an error condition to surface rather than a detail to resolve transparently. If a genuine redirect needs to be followed, that decision belongs to code that can evaluate the target — same host, expected path — before deciding whether the credential should go along with it, not to a general-purpose HTTP client's default behavior.

## 💡 Takeaway

- **A convenience default in an HTTP client is a security decision you didn't make on purpose.** Following redirects transparently is fine for a browser and wrong for anything holding a long-lived credential, and the library's default doesn't know which one you are.
- **The trigger for this class of bug is rarely malicious — it's a load balancer, a domain move, or a trailing slash.** That's what makes it easy to miss: nothing about the redirect looks suspicious, because nothing about it is, right up until the credential goes somewhere it shouldn't.
- **When you can't audit every redirect target a dependency might return, remove the client's ability to follow any of them automatically.** Fail loud on a 3xx and let calling code decide, rather than trusting the transport to make a judgment call it isn't equipped to make.
