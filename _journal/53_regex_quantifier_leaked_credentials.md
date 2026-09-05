---
title: "One Character in a Redaction Regex Let Passwords Through in Plaintext"
collection: journal
permalink: /journal/one-character-regex-leaked-credentials/
excerpt: "A regex meant to strip credentials out of connection-string-style URLs before they reached a logging or API layer worked for every URL with a username, and passed a password-only URL straight through untouched — one quantifier away from the case it was supposed to catch."
---

**The trick:** when a redaction pattern assumes a field is always present, test it specifically against the case where that field is empty or absent. A `+` (one or more) instead of a `*` (zero or more) is the entire difference between "always redacts" and "redacts only when the input looks like the common case."

## Issue

A regex responsible for stripping credentials out of URLs before they were logged or sent onward had been in place for some time, apparently working, when a review turned up URLs of a specific shape passing through completely unredacted — password and all.

## Root Cause

The pattern was built around the assumption that a credential-bearing URL always has the shape `username:password@host` — and required **at least one character** for both the username and password portions to match. A connection string of the form `scheme://:password@host` — password only, empty username, a completely standard and common way to write such a URL — has zero characters where the pattern demanded one or more. The regex simply didn't match, the redaction step passed the string through unchanged, and the plaintext password reached whatever logging or downstream API call came next.

Every URL that happened to include a username matched perfectly and redacted correctly, for as long as anyone tested it that way. The gap was invisible unless you specifically constructed the one input shape the pattern's author hadn't pictured: the credential half being present, and the identity half being empty.

## Solution

Change the username portion's quantifier from requiring at least one character to allowing zero, matching what the URL format actually permits rather than what the common case looks like. One character, `+` to `*`, closes the gap completely.

## 💡 Takeaway

- **A redaction, validation, or sanitization pattern that assumes a field is always non-empty needs a test case where that field is empty.** Assumption-driven regexes fail exactly on the input the assumption excludes, and that input is often perfectly valid by the format's own spec, not an edge case anyone invented.
- **A quantifier is not a detail.** The difference between `+` and `*` in a security-relevant pattern is the difference between "matches the common shape" and "matches every shape the format actually allows" — and an attacker, or just an unusual but valid input, doesn't have to try hard to find the gap.
- **Anything that touches credentials — redaction, masking, scrubbing before a log line — deserves its own focused test suite covering the format's actual grammar, not just the examples that were in mind when it was written.**
