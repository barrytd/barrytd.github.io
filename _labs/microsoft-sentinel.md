---
title: "Microsoft Sentinel - Cloud SIEM Setup and KQL Detection"
date: 2026-04-30
platform: Microsoft Azure
difficulty: Medium
category: blue-team
summary: "Self-directed Azure lab: stood up a Microsoft Sentinel workspace, connected threat intelligence data sources, and wrote KQL queries to detect suspicious sign-in patterns and IOC matches."
---

## Overview

A self-directed lab I built before a panel interview that asked for Microsoft cloud and SIEM experience.

**Microsoft Sentinel** is the cloud-native SIEM that runs on top of an **Azure Log Analytics workspace**. Detection logic is written in **KQL** (Kusto Query Language). The lab: spin up the workspace, connect data sources, write detection queries.

## Tools Used

- Azure portal
- Microsoft Sentinel
- Azure Log Analytics workspace
- KQL
- Built-in data connectors (Azure Activity, Azure AD, Threat Intelligence)
- Sentinel Content Hub

## Methodology

**Step 1 - Create the Log Analytics workspace.** Sentinel does not store data itself. It runs on top of a Log Analytics workspace, which is the actual data store. From the portal: *Create a resource → Log Analytics workspace*. Region and resource group matter for pricing and residency.

**Step 2 - Enable Sentinel on the workspace.** *Microsoft Sentinel → Add → select the workspace*. Sentinel is free for the first 31 days, then billed per GB ingested.

**Step 3 - Connect data sources.** Each connector is a pre-built integration. The ones I enabled:

- **Azure Activity** - subscription-level admin actions.
- **Azure Active Directory** - sign-in and audit logs.
- **Threat Intelligence Platforms** - TAXII / MISP feeds of known-bad IPs and domains.
- **Microsoft Defender Threat Intelligence** - Microsoft's curated TI feed.

**Step 4 - Failed sign-in detection.** KQL is pipe-delimited. Each operator runs against the output of the previous one.

```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"
| summarize FailedAttempts = count() by UserPrincipalName, IPAddress
| where FailedAttempts > 5
| order by FailedAttempts desc
```

- `SigninLogs` is the AAD table.
- `TimeGenerated > ago(24h)` bounds the scan to one day. Without it, the query scans the entire workspace history.
- `ResultType != "0"` excludes successes (0 means success).
- `summarize` groups and counts.
- `where FailedAttempts > 5` cuts the long tail.

This catches password spray and brute force against AAD.

**Step 5 - Threat-intel match.** With a TI connector enabled, the *ThreatIntelligenceIndicator* table holds known-bad IPs. Join sign-ins against it to find users authenticating from flagged addresses.

```kusto
let badIPs = ThreatIntelligenceIndicator
| where Active == true
| where NetworkIP != ""
| project NetworkIP, Description;
SigninLogs
| where TimeGenerated > ago(24h)
| join kind=inner (badIPs) on $left.IPAddress == $right.NetworkIP
| project TimeGenerated, UserPrincipalName, IPAddress, Description
```

`let` defines a reusable variable. `join kind=inner` returns only matches in both sides.

**Step 6 - Promote queries to analytics rules.** A query in *Logs* runs once. *Sentinel → Analytics → Create → Scheduled query rule* turns it into a continuous detection with a schedule, threshold, and entity mapping. Matching alerts are grouped into **incidents**, which is what an analyst works.

**Step 7 - Content Hub.** *Sentinel → Content hub* installs pre-built rule packs for AAD, Defender, Office 365, and dozens more. Reading the bundled KQL is a faster way to learn good detection patterns than writing from scratch.

## Key Takeaways

- Sentinel is Log Analytics with detection rules and SOC tooling on top. Knowing the workspace layer is the foundation.
- KQL transfers across Sentinel, Defender for Endpoint, Defender for Cloud, and Azure Resource Graph. Learning it well pays off everywhere in the Microsoft stack.
- Every query needs `| where TimeGenerated > ago(...)` on line two. Without it, the query scans the workspace history.
- Cloud SIEM is becoming the default. Sentinel, Splunk Cloud, Elastic Cloud, Chronicle. The setup-to-detection loop transfers between them.
- The Content Hub is the fastest study material. Read Microsoft's pre-built rules before writing your own.

## What a Defender Should Do

- Day-one connectors: Azure Activity and Azure AD logs. Those two catch most platform-level abuse.
- Connect at least one Threat Intelligence feed.
- Install the Azure Active Directory solution from the Content Hub for its pre-built rules. Tune noisy ones rather than disabling them.
- Automate the obvious cases with playbooks: confirmed phishing click triggers password reset and token revocation.
- Track ingestion volume. Sentinel is priced per GB ingested, and a noisy connector can produce a five-figure monthly bill. Set ingestion alerts.
- Pair Sentinel with Defender for Endpoint on Windows-heavy environments for cross-product correlation.
