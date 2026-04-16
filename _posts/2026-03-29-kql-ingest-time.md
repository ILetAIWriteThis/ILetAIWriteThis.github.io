---
layout: post
title: "KQL: When Did Your Logs Actually Arrive?"
category: serious-stuff
topic: KQL
---

There's a question that comes up more often than you'd think when working with log data: is this timestamp when the event *happened*, or when it *arrived*?

Turns out KQL has a function for that distinction -- `ingest_time()`.

## What is ingest_time()?

`ingest_time()` returns the time a record was ingested into the workspace. Not when the event occurred on the source system. Not when the agent sent it. When it actually landed in your tenant.

Your logs have two timestamps:

- **The event timestamp** -- the `Timestamp` field, representing when the thing actually happened on the source machine.
- **The ingest time** -- when the workspace received and stored the record.

Most of the time you don't care about the difference. But sometimes you really do.

## Why It Matters

I started looking at this because I wanted to understand ingestion lag -- how long it takes for log data to travel from a source system into my tenant and actually be queryable.

The query is simple:

```kql
TableName
| extend IngestDelay = ingest_time() - Timestamp
| summarize avg(IngestDelay), max(IngestDelay), min(IngestDelay) by bin(Timestamp, 1h)
```

This gives you the delta between when the event occurred and when it showed up in your workspace, bucketed by hour.

## What I Found

I tested this against Defender XDR tables. The general pattern is **3 to 5 minutes**, and it varies by table.

## The Practical Takeaway

If you're writing queries that depend on freshness, run this analysis on the specific tables you care about.

```kql
TableName
| where Timestamp > ago(24h)
| extend IngestDelay = ingest_time() - Timestamp
| summarize
    AvgDelay = avg(IngestDelay),
    P95Delay = percentile(IngestDelay, 95),
    MaxDelay = max(IngestDelay)
```

The `percentile(IngestDelay, 95)` column is the one you want to design around. Averages lie. P95 tells you what your worst normal case looks like.

Know your lag. Your queries will thank you.
