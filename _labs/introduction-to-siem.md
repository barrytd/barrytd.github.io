---
title: "Introduction to SIEM"
date: 2026-04-14
platform: TryHackMe
difficulty: Easy
category: blue-team
summary: "Triaged a cryptominer alert (cudominer.exe on HR_02 workstation), traced it to a 4688 process-creation detection rule on the *miner* keyword, and actioned it as a true positive."
---

## Overview

Introduction to SIEM is an easy TryHackMe room that walks through the **alert triage loop** every SOC analyst runs dozens of times per shift: *receive alert → confirm signal → identify scope → action → close ticket*.

The scenario: an alert fires for a host in the HR department for a process named *cudominer.exe*. The job is to confirm it's real, identify which detection rule fired, walk the audit data to make sure the rule didn't fire by accident, and pick the right triage action.

**SIEM** stands for *Security Information and Event Management*, the umbrella name for tools (Splunk, QRadar, Sentinel, Elastic Security) that ingest logs from across the environment, run detection rules against them, and surface alerts for analyst attention.

## Tools Used

- **The TryHackMe SIEM UI** (a simplified Splunk-style interface).
- **Windows EventID 4688** (process creation logs) as the core data source.
- **MITRE ATT&CK** as the reference framework.

## Methodology

**Step 1 - Read the alert carefully.** Every alert has the same skeleton: *what fired*, *what triggered it*, *on what host*, *for what user*, *when*. Reading all five before acting is the difference between a competent triage and a hasty close.

The alert in this room fires for **cudominer.exe** on host **HR_02** for user **HR_02$** (the machine account, which on Windows hosts is the user that runs services and scheduled tasks).

**Step 2 - Confirm the data behind the alert.** Click through to the underlying events. The 4688 record shows:

- New Process Name: `cudominer.exe`
- Process Command Line: typical cryptominer arguments (pool URL, wallet address, worker name).
- Parent Process: an unexpected parent (a user-space tool, not a service).

The data confirms the rule fired correctly. The process is genuinely a cryptominer.

**Step 3 - Identify the rule that fired.** The SIEM lets you click *"why did this alert fire?"* and see the rule definition. In this room, the rule is a simple keyword match on the substring **miner** in any process name.

That's a brittle rule (it would also catch *examiner.exe*, *miner.exe*, anything else with *miner* in the name) but it caught the real thing here. A more mature rule would match on process hash, parent process anomalies, or known cryptominer indicators.

**Step 4 - Pivot to other context.** A good analyst looks at *all* the data, not just the alert. The Windows event timeline for the host shows:

- *cudominer.exe* downloaded from an unusual URL.
- Persistence registry key added under HKCU\Software\Microsoft\Windows\CurrentVersion\Run.
- Outbound connections to a known mining pool URL.

All three are independent confirmation that the alert is real, not a false positive.

**Step 5 - Action the alert.** The triage actions in this room:

- **Raise as true positive** (correct here).
- **Close as false positive** (would be wrong; the data is unambiguous).
- **Escalate to incident response** (also reasonable for a confirmed compromise).
- **Suppress the rule** (would be wrong; the rule worked).

True positive raises the alert to the SOC team's incident queue for response.

## Key Takeaways

- **Triage is a loop, not a checkpoint.** Receive → confirm → pivot → action. Every alert goes through the same loop.
- **Read the data behind the alert.** A rule firing doesn't mean the rule was *right*. Confirm with the underlying events.
- **Look for independent confirmation.** Process creation alone isn't proof. Process + persistence + outbound connection is.
- **Don't suppress real findings.** A brittle rule that occasionally produces false positives is much better than no rule at all. Tune, don't disable.

## What a Defender Should Do

- Build detection rules around **multiple independent signals**, not single keyword matches. A cryptominer rule that requires *process name match + outbound connection to mining pool + persistence creation* will have near-zero false positive rate.
- Maintain a list of known-bad indicators (mining pool URLs, common cryptominer process names, persistence registry keys) and refresh it regularly. Threat intel feeds make this nearly automatic.
- Enable Windows EventID 4688 (Audit Process Creation) on every host and forward to the SIEM. It is the highest-value Windows event for SOC work.
- Block executables from running out of user-writable directories (Desktop, Downloads, AppData) with AppLocker or Windows Defender Application Control. Cryptominers and most commodity malware ship in those paths.
- Block outbound traffic to known mining pool ports/IPs at the egress. Most enterprise environments have no legitimate use for crypto mining traffic.
- Track *mean time to triage* and *mean time to action* as SOC metrics. Both should be measured in minutes, not hours.
