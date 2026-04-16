---
layout: post
title: "Getting Started with KQL"
category: serious-stuff
topic: KQL
---

So you want to learn KQL. Good choice. Kusto Query Language is what you use when you need to make sense of massive amounts of log data in Azure.

## The Basics

Here's your first query:

```
StormEvents
| where State == "TEXAS"
| count
```

That's it. Pipe operators. Filter. Count. KQL is surprisingly readable once you get the hang of it.

## Why KQL?

If you work with Azure Monitor, Log Analytics, Azure Data Explorer, or Microsoft Sentinel -- KQL is your bread and butter. It's fast, it scales, and it actually makes sense (most of the time).

## What's Next

More KQL posts coming. We'll get into `summarize`, `join`, `render`, and all the things that make your dashboards look impressive.
