---
title: "A Serializer Was Overridden on One Side, and the Test Codified the Bug"
collection: journal
permalink: /journal/a-serializer-override-is-a-pair/
excerpt: "After resuming a saved conversation state, a field written as plain text came back as a wrapper object, breaking every downstream string operation. The encoder had been overridden to compress the field. The decoder had not been taught to undo it — and the unit test asserted the broken shape as correct."
---

## Issue

After resuming an agent's saved state from a checkpoint, a field a node had written as ordinary text came back shaped differently — wrapped in an object instead of being the plain string it was written as. Every downstream operation expecting a string on that field broke.

## Root Cause

The checkpoint serializer had been extended to compress long string values before persisting them, wrapping each compressed string in a small sentinel envelope so the encoder could recognize its own output later. That half of the change was correct and complete.

The corresponding decode path was never extended to match. Serialization had become asymmetric by construction: writing a value now transformed it, and reading a value did not reverse the transformation. A round trip through the checkpoint store returned the wrapper, not the original string.

The part worth dwelling on is that the unit test covering this exact round trip **passed**, because it had been written by asserting on whatever the encoder actually produced rather than on what a correct round trip should return — it checked that the decoded value was the wrapper object, with the sentinel field set. The test was green, and it was testing the bug.

## Solution

Add the missing decode path: unwrap the sentinel and recursively walk any nested dicts, lists, and tuples for further wrapped values, so a compressed field buried inside a larger structure gets restored properly regardless of depth. Rewrite the test to assert on the property that actually matters — that the decoded value is a string equal to the original — rather than on the specific intermediate shape the buggy encoder happened to produce.

## 💡 Takeaway

- **A serializer override is a pair.** Overriding the encode half without the matching decode half doesn't fail loudly — it produces a coherent-looking format that nothing downstream expects.
- **Write round-trip tests as properties, not as golden values.** Asserting `decode(encode(x))` preserves the type and value of `x` catches this class of bug immediately. A test that pins the literal output of the encoder will happily bless whatever that encoder currently does, correct or not.
- **When you only see half of a symmetric operation change, ask where its inverse lives.** Any encode/decode, serialize/deserialize, or compress/decompress pair that gets touched on one side deserves an explicit check that the other side still agrees with it.
