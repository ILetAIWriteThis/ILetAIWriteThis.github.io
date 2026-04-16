---
layout: post
title: "Testing Defender for Cloud Alerts: Every Simulation Method Mapped"
category: serious-stuff
topic: Defender for Cloud
---

## TL;DR

Microsoft provides built-in simulation for 5 of 6 Defender for Cloud alert categories, but no single tool covers everything. Here's a complete map of every ready-to-execute test -- Microsoft's own methods, Atomic Red Team, Stratus Red Team, and MicroBurst -- so you know exactly what's covered and where the gaps are.

## Why You Should Care

You enabled Defender for Cloud. Alerts are configured. SIEM is connected. But have you actually seen one fire? If your alert pipeline breaks -- misconfigured connector, changed permissions, disabled plan -- you won't know until a real incident hits. Testing your detections before attackers do is the difference between a working SOC and an expensive dashboard.

## Microsoft's Built-In Simulation

Microsoft gives you three ways to generate test alerts, from one-click buttons to real attack simulations.

### Sample Alerts via Portal

Navigate to **Defender for Cloud -> Security Alerts -> Sample Alerts**, select your Defender plans (AppServices, DNS, KeyVaults, StorageAccounts, VirtualMachines), and click "Create sample alerts." Alerts appear in ~2 minutes, tagged with "Sample alert," and flow through to connected SIEMs and workflow automations. Requires Subscription Contributor role. This is your fastest pipeline validation.

### REST API Simulation

The same thing, but scriptable:

```
POST https://management.azure.com/subscriptions/{subscriptionId}/providers/Microsoft.Security/locations/{ascLocation}/alerts/default/simulate?api-version=2022-01-01

{
  "properties": {
    "kind": "Bundles",
    "bundles": ["AppServices", "DNS", "KeyVaults", "StorageAccounts", "VirtualMachines"]
  }
}
```

Supported bundles cover App Services, DNS, Key Vaults, Storage Accounts, and Virtual Machines (which generates Network Layer alerts). **API alerts have no dedicated bundle** -- a notable gap.

### Real-Alert Simulation Procedures

These trigger genuine detection logic, not synthetic samples:

| Category | Procedure | Alert Generated | Time to Alert |
|---|---|---|---|
| **App Service** | Browse to `https://<site>.azurewebsites.net/This_Will_Generate_ASC_Alert` | EICAR-style test alert | 1.5-4 hours |
| **API** | In API Management Test console, send request with User-Agent set to `javascript:` | Suspicious User Agent detected | Minutes to hours |
| **Key Vault** | Access Key Vault secrets via TOR Browser | Access from a TOR exit node (KV_TORAccess) | Minutes to hours |
| **Storage** | Upload EICAR test file; or generate blob SAS URL and download via TOR | Malicious file uploaded; Access from Tor exit node | 1-3 hours |
| **DNS** | Run `Resolve-DnsName` against known malicious domains from an Azure VM | Multiple DNS threat alerts | Minutes |
| **Network Layer** | No dedicated real-alert procedure exists | N/A -- sample alerts only | N/A |

Step-by-step playbooks live in Microsoft's GitHub Simulations repo at `github.com/Azure/Microsoft-Defender-for-Cloud/tree/main/Simulations`.

## DNS Simulation: The PowerShell Approach

DNS has the most actionable command-line test suite. Run these from any Azure VM using default Azure DNS:

```powershell
Resolve-DnsName kcp53.msupdate.us           # C2 communication alert
Resolve-DnsName all.mainnet.ethdisco.net     # Crypto mining alert
Resolve-DnsName micros0ft.com                # Suspicious domain alert
Resolve-DnsName 164e9408d12a701d91d206c6ab192994.info  # DGA-style alert
```

For DGA simulation, generate 1,000 random domain lookups:

```powershell
For($i=0; $i -le 1000; $i++) {
    $rand = -join ((97..122) | Get-Random -Count 63 | % {[char]$_})
    Resolve-DnsName "$rand.contoso.com" -ErrorAction Ignore
}
```

**Timing gotchas:** New Defender for DNS plans need **2 hours** before alerts activate. Newly created VMs need **4 hours**. Repeat queries on the same VM won't trigger new alerts unless you run `ipconfig /flushdns`.

## Atomic Red Team: 8 Verified Azure Tests

Confirmed Azure-specific tests from the Atomic Red Team master branch:

### Storage (4 tests -- best covered, forms a complete attack chain)

| Test | MITRE Technique | What It Does |
|---|---|---|
| T1619 #2 | Cloud Storage Object Discovery | Enumerates storage accounts, lists file shares, containers, blobs, tables, queues |
| T1619 #3 | Cloud Storage Object Discovery | Tests for unauthenticated public access to blob containers |
| T1619 #4 | Cloud Storage Object Discovery | Brute-forces container and blob names using MicroBurst |
| T1530 #2 | Data from Cloud Storage | Downloads all file share and blob content |

These chain together: T1619 discovers targets, T1530 exfiltrates. Can trigger `Storage.Blob_AccessInspectionAnomaly`, `Storage.Blob_DataExfiltration`, and `Storage.Blob_AnonymousAccessAnomaly`.

### App Service (2 tests -- indirect coverage)

| Test | MITRE Technique | What It Does |
|---|---|---|
| T1528 #1 | Steal Application Access Token | Uploads tampered zip to Function's blob storage for RCE and identity token theft |
| T1528 #2 | Steal Application Access Token | Modifies Function source code via File Share to exfiltrate managed identity tokens |

### DNS (4 Windows-based tests)

T1071.004 atomics include DNS Large Query Volume, Regular Beaconing, Long Domain Query (tunneling), and C2 via dnscat2. These are Windows PowerShell tests -- run them on Azure VMs to trigger Defender for DNS.

**Key Vault has no dedicated Atomic Red Team test.** A proposed test (GitHub Issue #3043) was closed without merging.

## Stratus Red Team: Azure Storage Focus

DataDog's Stratus Red Team offers **10 Azure attack techniques**, with 4 directly targeting Storage:

| Technique | Tactic | Category |
|---|---|---|
| `azure.exfiltration.storage-public-access` | Exfiltration | Storage |
| `azure.exfiltration.storage-sas-export` | Exfiltration | Storage |
| `azure.impact.blob-ransomware-client-encryption-scope` | Impact | Storage + Key Vault |
| `azure.impact.blob-ransomware-individual-file-deletion` | Impact | Storage |
| `azure.exfiltration.disk-export` | Exfiltration | General |
| `azure.execution.vm-custom-script-extension` | Execution | General |
| `azure.execution.vm-run-command` | Execution | General |
| `azure.persistence.create-bastion-shareable-link` | Persistence | General |
| `azure.privilege-escalation.root-user-access-administrator` | Privilege Escalation | General |
| `azure.impact.resource-lock` | Impact | General |

Stratus is a Go binary that uses Terraform to provision resources, execute attacks, and clean up automatically. The `blob-ransomware-client-encryption-scope` technique is notable -- it creates an encryption scope using a Key Vault key, which can indirectly trigger Key Vault alerts. **No coverage** for App Service, API, DNS, or Network Layer.

## MicroBurst: The Key Vault Answer

NetSPI's MicroBurst PowerShell toolkit fills the Key Vault gap. Defender for Cloud **explicitly detects MicroBurst** and generates named alerts when its functions run:

- **`Get-AzPasswords`** -- Dumps credentials from Key Vaults, App Service configs, Automation accounts. Triggers `KV_OperationVolumeAnomaly`, `KV_ListGetAnomaly`, and Resource Manager alerts specifically naming MicroBurst.
- **`Get-AzureKeyVaults-Automation`** -- Dumps Key Vault secrets via Automation Accounts. Similar alerts.
- **`Invoke-EnumerateAzureBlobs`** -- Brute-forces storage container names. Already used by Atomic Red Team T1619 #4.
- **`Invoke-EnumerateAzureSubDomains`** -- Enumerates Azure subdomains via DNS.
- **`Get-AzDomainInfo`** -- Comprehensive subscription enumeration.

## Coverage Matrix

| Tool / Method | App Service | API | Key Vault | Storage | DNS | Network Layer |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Portal Sample Alerts** | Y | ~ | Y | Y | Y | Y |
| **REST API Simulate** | Y | N | Y | Y | Y | Y* |
| **MS Real-Alert Procedures** | Y | Y | Y | Y | Y | N |
| **Atomic Red Team** | ~ | N | N | YY | ~ | N |
| **Stratus Red Team** | N | N | ~ | YY | N | N |
| **MicroBurst** | ~ | N | YY | Y | ~ | N |

YY = strong multi-test coverage, Y = direct coverage, ~ = indirect/partial, N = none, *via VirtualMachines bundle

**Well covered:** Storage (every framework), Key Vault (MicroBurst + TOR procedure), DNS (PowerShell scripts + Atomic Red Team).

**Gaps:** App Service relies on Microsoft's test URL and two Atomic tests. API only has Microsoft's User-Agent trick. Network Layer has zero real-alert simulation -- sample alerts only.

## Takeaway

Layer your testing: **first** use the REST API or Portal for sample alerts across all six categories to validate your pipeline. **Second**, run Microsoft's real-alert procedures (TOR for Key Vault/Storage, test URL for App Service, `Resolve-DnsName` for DNS) to confirm detection logic fires. **Third**, supplement with Atomic Red Team's Storage chain, Stratus Red Team's storage attacks, and MicroBurst's `Get-AzPasswords` for realistic scenarios. The biggest blind spots are Network Layer (sample alerts only) and API (Microsoft's test console only) -- plan accordingly.
