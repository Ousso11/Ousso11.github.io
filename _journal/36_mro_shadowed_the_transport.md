---
title: "A Method Name Collision Broke Requests on the Async Path Only"
collection: journal
order: 16
permalink: /journal/method-override-with-a-different-signature/
excerpt: "An async client call raised TypeError about unexpected arguments, only on the async path. A subclass had defined a method with the exact same name as a base-class transport method but a completely different signature, and a type: ignore comment was sitting right on top of it."
---

**The trick:** never let a subclass method share a name with a base-class method it isn't meant to override — and never suppress a type checker's override-incompatibility warning without reading what it's telling you first.

## Issue

An async client call failed with a `TypeError` about unexpected arguments. The identical synchronous call worked perfectly.

## Root Cause

The base transport class and a subclass built on top of it both defined a method with the same name — one generic, one domain-specific, with incompatible signatures. Python's method resolution order made the subclass win. When the transport's own internals called what they thought was their own method, they dispatched into the subclass instead and blew up on arity. A type-checker suppression comment sat directly on the override, silencing the one tool that would have caught it.

## Solution

Rename the subclass method so it no longer collides, and remove the suppression.

## 💡 Takeaway

- A `# type: ignore[override]` on an incompatible signature is a live dispatch bug waiting to happen, not a lint nit.
- Never let a domain method share a name with a transport method in the same hierarchy.
- A bug on only one of two near-identical paths (sync vs async) is often a naming collision specific to that path.
