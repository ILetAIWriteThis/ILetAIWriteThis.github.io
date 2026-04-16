---
layout: post
title: "Detecting WDAC Policy Abuse Before Your EDR Goes Silent"
category: serious-stuff
topic: WDAC
---

## TL;DR

Attackers can deploy a Windows Defender Application Control (WDAC) deny policy to kill your EDR. Here's how WDAC policies get deployed, what artifacts they leave behind, and the KQL you need to catch it.

## Why You Should Care

WDAC was designed to control which applications and drivers are allowed to run. But a crafted deny policy can block your security tooling -- EDR, AV, the works. An attacker with local admin drops a policy, forces a refresh, and your endpoint goes dark. No alerts, no telemetry, nothing. If you're not watching for policy changes, you won't know until it's too late.

## How WDAC Policies Get Deployed

Understanding the deployment mechanisms is key to knowing what to look for. There are several ways a WDAC policy gets onto a system:

### Legacy: SIPolicy.p7b

The classic method. A signed binary policy file dropped into:

```
C:\Windows\System32\CodeIntegrity\SIPolicy.p7b
```

This is the single-policy format used on older Windows versions. One file, one policy. A reboot activates it. Simple and well-documented -- which means attackers know it too.

### PowerShell: New-CIPolicy and Related Cmdlets

The `ConfigCI` PowerShell module gives you everything you need to create and manage policies:

```powershell
# Create a new policy
New-CIPolicy -Level Publisher -FilePath "C:\policy.xml" -UserPEs

# Convert to binary
ConvertFrom-CIPolicy -XmlFilePath "C:\policy.xml" -BinaryFilePath "C:\policy.p7b"

# Merge policies
Merge-CIPolicy -PolicyPaths "C:\base.xml","C:\deny.xml" -OutputFilePath "C:\merged.xml"
```

Any use of these cmdlets outside of a known change window is suspicious. Full stop.

### Modern: CiTool.exe

On newer Windows builds (Windows 11 22H2+, Server 2025), `CiTool.exe` is the official way to manage multiple simultaneous policies:

```powershell
# Deploy a policy
CiTool.exe --update-policy "C:\policies\{GUID}.cip"

# List active policies
CiTool.exe --list-policies

# Remove a policy
CiTool.exe --remove-policy "{policy-guid}"
```

The multi-policy format stores individual `.cip` files as GUIDs:

```
C:\Windows\System32\CodeIntegrity\CiPolicies\Active\{GUID}.cip
```

### RefreshPolicy.exe

On systems that don't have `CiTool.exe`, `RefreshPolicy.exe` forces a policy reload without reboot:

```powershell
RefreshPolicy.exe
```

This is a red flag if seen in the wild. Legitimate deployments through MDM or Group Policy don't typically need manual refreshes.

## Detection Strategy: Union of Signals

No single event tells the full story. You need to combine multiple signals. Here's the approach:

### Signal 1: Policy Loaded Events

Microsoft Defender for Endpoint tracks `CodeIntegrityPolicyLoaded` events. When a new or modified WDAC policy activates, this fires. Unexpected policy loads -- especially outside of deployment windows -- are your first indicator.

### Signal 2: File Creation in CodeIntegrity Folders

Watch for file writes to:

- `C:\Windows\System32\CodeIntegrity\SIPolicy.p7b`
- `C:\Windows\System32\CodeIntegrity\CiPolicies\Active\*.cip`

Any new or modified file in these paths that doesn't correlate with a known Intune/GPO deployment is worth investigating immediately.

### Signal 3: Suspicious Tool Usage

PowerShell cmdlets (`New-CIPolicy`, `ConvertFrom-CIPolicy`, `Merge-CIPolicy`, `Set-CIPolicyIdInfo`) and binaries (`CiTool.exe`, `RefreshPolicy.exe`) executing on endpoints where WDAC isn't being actively managed.

### Signal 4: Event Code 3099

This is your SIEM workhorse. **Windows Event ID 3099** (Microsoft-Windows-CodeIntegrity/Operational) logs when a Code Integrity policy is loaded or changed. Forward this to your SIEM and alert on it.

```
Event ID 3099: A Code Integrity policy has been loaded.
Log: Microsoft-Windows-CodeIntegrity/Operational
```

If you're collecting Windows event logs in Sentinel or any SIEM, this event should be part of your baseline detections.

## KQL: Defender XDR Detection

This query runs in Microsoft Defender XDR Advanced Hunting. It unions three signal types from native Defender tables and alerts when multiple signals converge on the same device:

```kql
let timeframe = 24h;
// Signal 1: Code Integrity policy loaded events
let PolicyLoaded = DeviceEvents
| where Timestamp > ago(timeframe)
| where ActionType == "CodeIntegrityPolicyLoaded"
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, InitiatingProcessFileName,
          InitiatingProcessCommandLine, ReportId
| extend SignalType = "PolicyLoaded";
// Signal 2: Suspicious file creation in CodeIntegrity directories
let FileCreated = DeviceFileEvents
| where Timestamp > ago(timeframe)
| where FolderPath matches regex @"(?i)\\CodeIntegrity\\(SIPolicy\.p7b|CiPolicies\\Active\\.*\.cip)"
| where ActionType in ("FileCreated", "FileModified")
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, InitiatingProcessFileName,
          InitiatingProcessCommandLine, ReportId
| extend SignalType = "FileInCodeIntegrityFolder";
// Signal 3: WDAC tooling usage (PowerShell cmdlets and CiTool)
let ToolUsage = DeviceProcessEvents
| where Timestamp > ago(timeframe)
| where (FileName =~ "CiTool.exe" and ProcessCommandLine has_any ("--update-policy", "--remove-policy"))
    or (FileName =~ "RefreshPolicy.exe")
    or (FileName =~ "powershell.exe" and ProcessCommandLine has_any
        ("New-CIPolicy", "ConvertFrom-CIPolicy", "Merge-CIPolicy",
         "Set-CIPolicyIdInfo", "Set-RuleOption"))
| project Timestamp, DeviceName, ActionType = "ToolExecution", FileName, FolderPath = "",
          InitiatingProcessFileName = InitiatingProcessFileName,
          InitiatingProcessCommandLine = ProcessCommandLine, ReportId
| extend SignalType = "WDACToolUsage";
// Union all signals
PolicyLoaded
| union FileCreated, ToolUsage
| sort by DeviceName, Timestamp asc
| summarize Signals = make_set(SignalType), EventCount = count(),
            FirstSeen = min(Timestamp), LastSeen = max(Timestamp),
            Details = make_set(InitiatingProcessCommandLine)
    by DeviceName
| where array_length(Signals) >= 2  // Alert when multiple signal types converge
```

This lights up when **two or more** signal types fire on the same device -- significantly reducing false positives from legitimate policy management while catching actual abuse.

## KQL: SIEM Detection via Event ID 3099

If you're forwarding Windows Event Logs to Microsoft Sentinel or another SIEM, **Event ID 3099** from the `Microsoft-Windows-CodeIntegrity/Operational` log is your best standalone signal. This event fires every time a Code Integrity policy is loaded or changed -- no Defender XDR required.

```kql
SecurityEvent
| where EventID == 3099
| where Channel == "Microsoft-Windows-CodeIntegrity/Operational"
| project TimeGenerated, Computer, EventData
| extend PolicyFile = extract(@"PolicyFileName=(.+?);", 1, EventData)
| where PolicyFile !in ("expected_policy_1.p7b", "expected_policy_2.cip")  // Whitelist your known policies
```

Whitelist your legitimate policies and alert on everything else. If you're not collecting this event yet, add it to your Windows Event Log collection rules -- it's low volume and high value.

## Takeaway

Add WDAC policy monitoring to your detection stack now -- not after an attacker uses it to blind your EDR. Collect Event ID 3099, watch the CodeIntegrity folders, and alert on unexpected tool usage. The union approach across multiple signals gives you high-confidence detections with minimal noise. If someone is touching WDAC on your endpoints and it's not your endpoint management team, you have a problem.
