---
title: "Investigating with Splunk"
date: 2026-03-10
platform: TryHackMe
difficulty: Medium
category: blue-team
summary: "Multi-host compromise investigation: WMI backdoor creation, decoded a double-encoded PowerShell Empire C2 beacon, and traced lateral movement through Sysmon and Windows event correlation."
---

## Overview

Investigating with Splunk is a medium-difficulty TryHackMe room that simulates a **multi-host enterprise compromise** investigation. The data is Sysmon plus standard Windows security event logs, ingested into Splunk.

The chain to uncover: an initial workstation is compromised, the attacker uses **WMI** (Windows Management Instrumentation) to create a backdoor, **PowerShell Empire** C2 beacons are sent from the compromised host (double-encoded to evade naive detection), and the attacker laterally moves to other hosts using the captured credentials.

## Tools Used

- **Splunk** for SPL queries.
- **Sysmon** as the primary data source (much richer than default Windows logs).
- **CyberChef** for decoding the encoded PowerShell payload.
- **MITRE ATT&CK** for technique mapping.

## Methodology

**Step 1 - Establish the timeline.** The first SOC question is always *when did this start?* A frequency-by-hour graph of all events on the alerted host narrows the active window.

```spl
index="sysmon" host="VICTIM_01" 
| timechart span=1h count
```

A spike at a specific hour identifies the starting point.

**Step 2 - Identify the initial vector.** Within the spike window, look for the *first* unusual process. Sysmon EventID 1 (Process Create) is the workhorse.

```spl
index="sysmon" host="VICTIM_01" EventCode=1 
| reverse 
| table _time, Image, ParentImage, CommandLine
```

The earliest unusual entry might be *winword.exe* spawning *powershell.exe*, which is the classic Office macro → PowerShell pattern (MITRE T1059.001).

**Step 3 - Decode the PowerShell payload.** The command line of the spawned PowerShell shows a *long base64-encoded blob* with the `-EncodedCommand` flag. That's the standard PowerShell technique for hiding payloads in command lines.

The encoded payload here was *double-encoded*: base64-decoding it produces *another* base64 blob, which decodes to the actual PowerShell that fetches and runs the Empire stager.

```python
# in CyberChef or Python:
import base64
once = base64.b64decode(blob).decode('utf-16le')
twice = base64.b64decode(once).decode('utf-16le')
print(twice)
```

The `utf-16le` decode is required because PowerShell's *EncodedCommand* expects UTF-16 little-endian internally.

**Step 4 - Identify the C2 server.** The decoded payload contains a URL: the attacker's PowerShell Empire C2 server. *Empire* is an open-source post-exploitation framework with a distinctive beacon pattern (periodic HTTP requests to a fixed URI with attacker-encoded data in the headers).

**Step 5 - Find lateral movement.** WMI is the attacker's preferred lateral movement tool because it's built into Windows and rarely logged in detail. Sysmon EventID 21 (WMI EventConsumerToFilter) and EventCode 22 (DNS query) help.

```spl
index="sysmon" EventCode IN (19,20,21) 
| table _time, host, User, Operation
```

A WMI permanent event subscription created on a different host is the backdoor: it listens for a specific event (e.g., system startup) and runs an attacker payload when it fires.

**Step 6 - Catalog the indicators.** By the end, the case file has:

- The initial vector (Office macro → PowerShell).
- The C2 URL and infrastructure.
- The WMI backdoor (event subscription on the secondary host).
- The lateral movement path (credential reuse from victim_01 to victim_02 via WMI).
- IOC list to add to detection (Empire beacon URI pattern, the encoded payload hash, the C2 domain).

## Key Takeaways

- **Sysmon is the difference between meaningful Windows logging and useless Windows logging.** Default Windows audit policy captures a fraction of what Sysmon does. Sysmon's process tree visibility is essential.
- **PowerShell encoded commands are an automation flag, not a malicious flag, but the combination of long encoded commands + unusual parent processes is high-confidence malicious.**
- **WMI is the *quiet* lateral movement channel.** It's enabled by default, runs as SYSTEM, and is rarely logged. Sysmon EventIDs 19/20/21 are the only mature detection.
- **Double-encoded payloads are common enough that decoders need to handle recursion.** CyberChef handles this with a *Magic* operation.

## What a Defender Should Do

- Deploy Sysmon to every Windows endpoint with a tuned config (Olaf Hartong's *sysmon-modular* is the standard starting point).
- Forward Sysmon logs to the SIEM. Without forwarding they're useless.
- Block Office macros from running for all users by Group Policy. There is almost no legitimate use of macros in modern enterprises, and macros are the most common initial-vector vehicle.
- Detect the PowerShell *EncodedCommand* + unusual-parent pattern as a high-confidence rule.
- Audit WMI event subscriptions regularly. Any subscription on a non-admin endpoint is a finding. The Get-WMIObject query against __EventFilter, __FilterToConsumerBinding, and __EventConsumer enumerates them.
- Block known C2 frameworks at the egress: Empire has distinctive URIs and User-Agent strings, the same is true for Cobalt Strike, Sliver, and Mythic.
- Tier the admin model so a compromised workstation cannot harvest credentials that work on other hosts.
