---
title: "Two Methods Shared One Name. Only One of Them Worked."
collection: journal
permalink: /journal/method-override-with-a-different-signature/
excerpt: "An async client call raised TypeError about unexpected arguments, only on the async path. A subclass had defined a method with the exact same name as a base-class transport method but a completely different signature, and a type: ignore comment was sitting right on top of it."
---

**The trick:** never let a subclass method share a name with a base-class method it isn't meant to override — and never suppress a type checker's override-incompatibility warning without reading what it's telling you first.

## Issue

An async client call failed with a `TypeError` about unexpected arguments. The equivalent synchronous call worked perfectly. Nothing in the immediate call site looked wrong.

## Root Cause

The base transport class defined a generic async request method taking a method name, an endpoint, and a JSON body. A subclass, built for a higher-level API on top of that transport, defined its own method — with the **same name** — taking a single structured request object instead.

Python's method resolution order means the subclass definition wins unconditionally. When the generic transport's own internal code called what it believed was its own method, passing the three original arguments, it was actually dispatching into the subclass's differently-shaped method and blowing up on arity.

The override had a type-checker suppression comment sitting directly on top of it. The type checker had, correctly, flagged an incompatible override — and the suppression silenced the one piece of tooling that would have caught this before it ran.

## Solution

Rename the subclass method so it no longer collides with the base class's name, and remove the suppression that had been hiding the warning.

## 💡 Takeaway

- **A type-checker suppression on an incompatible override is not a lint nit — it's a live dispatch bug waiting for the base class to call itself.** Treat `# type: ignore[override]` as a note to come back and rename something, not a note to move on.
- **In a class hierarchy where the base is "the transport" and the subclass is "the domain API," never let a domain method share a name with a transport method.** The collision is invisible until the transport's own internals try to call themselves and land in the wrong method instead.
- **A bug that only appears on one of two nearly identical code paths (sync vs. async, one provider vs. another) is often a naming collision specific to that path**, not a logic error shared by both.
