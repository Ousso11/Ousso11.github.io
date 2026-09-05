---
title: "Every Number Was a Little Low, and Nothing Ever Failed"
collection: journal
order: 20
permalink: /journal/empty-is-not-the-same-as-failed/
excerpt: "Usage events drained from a Redis Stream into the billing table. A helper returned an empty list on any exception, and the caller read emptiness as permission to XTRIM the stream to zero. Dashboards and invoices were quietly low, with no error anywhere."
---

**The trick:** trim a durable queue to what's been acknowledged (`MINID`), never to a fixed count (`MAXLEN`) — and never let "I read nothing" become indistinguishable from "there was nothing to read."

## Issue

Dashboards and invoices were quietly low for weeks. No error, no alert, no failed request — just numbers smaller than they should have been.

## Root Cause

A background job drained a Redis Stream into the billing table. Its reader returned an empty list on *any* exception, not just an empty stream — and the caller treated "read nothing" as "stream is stale," running `XTRIM MAXLEN 0` and permanently deleting hundreds of unbilled events on a single Redis hiccup. A second bug trimmed to a fixed count after every ack, capping throughput before the system started eating its own data.

## Solution

Delete the trim-to-zero branch entirely — no situation justifies it. Trim only up to what's been acknowledged (`XTRIM MINID`), never by count. Give every event a unique id so retries can't double-count either.

## 💡 Takeaway

- "No data" and "couldn't read the data" must never collapse into the same value.
- Destructive cleanup on a queue should be expressed relative to progress, never to a count.
- Fail-open locking is a choice about the *job*, not the lock — money and side effects want fail-closed.
