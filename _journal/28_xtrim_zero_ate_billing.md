---
title: "A Redis Read Error Deleted Every Unbilled Event"
collection: journal
permalink: /journal/empty-is-not-the-same-as-failed/
excerpt: "Usage events drained from a Redis Stream into the billing table. A helper returned an empty list on any exception, and the caller read emptiness as permission to XTRIM the stream to zero. Dashboards and invoices were quietly low, with no error anywhere."
---

## Issue

Nothing. That is the story.

Dashboards were low. Monthly totals were low. Enterprise invoices were low. There was no error, no alert and no failed request — just numbers that were smaller than they should have been, with nothing to compare them against.

## Root Cause

Usage events land in a Redis Stream and a background job drains them into the usage table, which is ground truth for both dashboards and invoices. The drain logic reduced to:

```python
aggregated = read_and_aggregate()
if not aggregated and stream_len > 0:
    trim_stream(max_len=0)      # "stream is stale, clear it"
```

The reader returned `[]` on **any caught exception**, not only on an empty stream. So one Redis hiccup during the read produced `aggregated == []` alongside a stream length of several hundred — and the "cleanup" branch executed `XTRIM MAXLEN 0`, permanently deleting hundreds of unbilled events.

A second bug sat right below it: after acknowledging, the code trimmed to a fixed length of 1000. That is a hard ceiling on throughput before the system starts eating its own billing data.

And a third, in the same job. The flush inserted, then acknowledged, under a distributed lock that **failed open**, with a **shared consumer name** reading its own pending list from the beginning. A slow insert outliving the lock let a second instance re-read the same pending entries and insert them again — every metric doubled, invoices included. The rows carried no unique event key, so the database had no way to dedupe them either. The lock release deleted the key unconditionally, so a slow holder would delete *someone else's* lock on the way out.

## Solution

**Delete the trim-to-zero branch entirely.** There is no situation in which "I read nothing" justifies destroying data.

**Express the post-acknowledge trim in terms of what was acknowledged**, not how many items to keep: `XTRIM MINID <last_acked_id>`. You can never trim past what you have acknowledged, at any volume. `MAXLEN` is a capacity primitive; `MINID` is a correctness primitive.

**Give every instance its own consumer name**, built from hostname, process id and a random suffix.

**Make lock release a token-checked compare-and-delete** in Lua, and make lock acquisition **fail closed** on error rather than returning success.

**Give every event an identity.** A client-generated event id, a unique index, and insert-on-conflict-do-nothing — so the same identity covers the queue, the database insert and the replay path, with a dead-letter queue for rows that can never be accepted.

## 💡 Takeaway

- **"No data" and "could not read the data" must never be the same value.** A helper that swallows exceptions and returns an empty list will eventually be consumed by code that reads emptiness as permission to destroy.
- **Destructive cleanup in a queue must be expressed relative to what was acknowledged.** Any trim, prune or vacuum bounded by *count* rather than *progress* will discard unprocessed work under load.
- **Fail-open locking is a choice about the job, not about the lock.** For anything with an external side effect or a money implication, failing closed is the only correct default.
- **Idempotency needs an identity you control.** Without a unique event key on the row, at-least-once delivery is indistinguishable from double billing.
