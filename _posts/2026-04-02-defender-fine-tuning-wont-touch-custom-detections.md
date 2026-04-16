---
layout: post
title: "Defender Fine-Tuning Won't Touch Your Custom Detections"
category: serious-stuff
topic: Microsoft Defender
---

## TL;DR

Alert suppression via Defender's fine-tuning page only applies to Microsoft-built alerts — custom detection rules are silently ignored, and suppressed alerts are not exposed through the Graph API.

## Why You Should Care

If you're managing alert noise in an environment with custom KQL detection rules, you might be configuring suppression in the wrong place entirely. The fine-tuning page looks like it covers everything. It doesn't. And if your SIEM or automation pipeline pulls alerts via Graph API, suppressed built-in alerts won't show up there either — which means your suppression logic is invisible to your pipeline and your custom alerts are unfiltered regardless.

## What Actually Happens

Defender's fine-tuning / alert suppression page operates on Microsoft-managed alerts only. When you suppress an alert there:

- **MS-built alerts** → suppressed as expected, hidden in the portal
- **Custom detection alerts** → suppression has no effect, alerts fire normally
- **Graph API** → suppressed MS alerts are not returned, but this behavior is not surfaced clearly in the portal or docs
- **Streaming API** → suppressed alerts *are* returned, even when hidden in the portal — suppression does not filter them out here

The silent part is the dangerous part. There's no error. No warning. It just doesn't work for custom rules and doesn't tell you.

## The Right Approach

For custom detection rules, suppression logic belongs in the **query itself**, not on the suppression page. If you want to exclude a specific condition — a known-good process, a test endpoint, a suppressed CI pipeline — filter it in KQL before the alert fires.

```kql
// Instead of suppressing after the fact:
DeviceProcessEvents
| where FileName == "suspicious.exe"
| where DeviceName !in ("test-machine-01", "build-agent-02")  // filter here
```

Fine-tuning page = noise reduction for Microsoft's own signal.
Custom detections = you own the logic, tune it in the query.

## Takeaway

Don't trust the suppression UI to be aware of your custom rules. Different behavior on different APIs