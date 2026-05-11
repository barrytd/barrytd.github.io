---
title: "ItsyBitsy"
date: 2026-05-10
platform: TryHackMe
difficulty: Easy
category: blue-team
summary: "An IDS alert from an HR workstation turns into a hunt for an unusual user-agent in Kibana, ending on a Pastebin paste being used as a tiny command-and-control channel."
---

## Overview

ItsyBitsy is an easy TryHackMe room built around a realistic SOC scenario: an IDS rule fires for a workstation in the **HR** department with a vague *"potential C2 communication"* alert, and the analyst has to confirm whether it's real, identify what happened, and recommend next steps.

The investigation runs entirely inside **Kibana**, the search and dashboard front-end of the **ELK stack** (*Elasticsearch + Logstash + Kibana*). The trail starts with 1,482 HTTP connection records, narrows to one with a suspicious *user-agent*, and ends on a Pastebin URL being used as a low-volume C2 channel by an attacker who was *living off the land*. The takeaway is the technique, not the room flag.

## Tools Used

- **Kibana Discover** for searching and filtering the *connection_logs* index.
- **Elasticsearch query bar** (Lucene syntax) for the actual filter.
- **A browser** to visit the recovered Pastebin URL.
- **MITRE ATT&CK** as the reference framework for naming the technique.

## Methodology

**Step 1 - Scope the time window.** The first move in any timeline-driven investigation is to *narrow the time range*. The alert names March 2022, so Kibana's time range is set to *Mar 1, 2022 to Mar 31, 2022*. 1,482 events total, small enough to scan field by field instead of building heavy aggregations.

**Step 2 - Pick the right field to look at.** When you do not know what you are looking for, the smartest move is to look at fields with *low natural variety* in the data. HTTP traffic in a healthy environment has surprisingly few distinct user-agent strings: Chrome, Firefox, Edge, maybe an Office update agent. Anything else is unusual.

**Step 3 - Find the anomalous record.** Scrolling the per-event view (or filtering directly on the *user_agent* field) surfaces a single record whose user-agent reads *bitsadmin*. That is not a browser. That is the command-line front-end of **BITS** (*Background Intelligent Transfer Service*), the built-in Windows service that downloads files in the background for Windows Update and similar tooling.

**Step 4 - Map to MITRE ATT&CK.** bitsadmin in a user-agent field is a textbook **LOLBin** (*living off the land binary*: a trusted built-in OS tool being repurposed for attacker activity). The mappings:

- **T1197 - BITS Jobs** is the exact technique of using bitsadmin to fetch a payload.
- **T1105 - Ingress Tool Transfer** is the umbrella for "malware retrieves more tooling from the internet."
- **T1071.001 - Application Layer Protocol: Web Protocols** covers the choice of plain HTTP to a popular destination.

**Step 5 - Confirm the destination.** The same record shows the host connecting outbound to **pastebin.com** over port 80, with method HEAD (used by bitsadmin to check the file before downloading it). Pastebin is a legitimate text-hosting service that real developers use daily, which means most corporate proxies allow it. That is exactly why attackers love it. A small HEAD or GET to a Pastebin URL on a schedule is indistinguishable from a developer checking a colleague's paste.

**Step 6 - Build a literal detection.** The Kibana Lucene query that would catch this single event in real time is small:

```
user_agent : "bitsadmin" AND host : "pastebin.com"
```

A more general production version broadens both axes:

```
user_agent : ("bitsadmin" OR "curl" OR "wget" OR "python-requests" OR /PowerShell.*/)
  AND host : (/pastebin\.com/ OR /paste\.ee/ OR /ghostbin\.com/ OR /hastebin\.com/)
```

The user-agent allow-list ages well. The paste-site list is the brittle part because attackers rotate constantly.

**Step 7 - Visit the URL to confirm payload retrieval.** Loading the recovered Pastebin URL in a sandboxed browser shows a *secret.txt* paste containing attacker-staged content. This is the *stage two* retrieval, the moment the analyst confirms the host actually pulled down attacker-controlled content rather than just performing a benign check.

**Step 8 - Plan the real-world pivots.** A real SOC investigation would treat the recovered URL as the *start* of the case, not the end:

- Ask EDR for the *parent process* of bitsadmin.exe on the workstation at the timestamp from the log. bitsadmin launched by *winword.exe* or by a script host is way more interesting than bitsadmin launched by a real admin in cmd.exe.
- Correlate the Zeek *uid* (the connection identifier) back to the parent *conn.log* entry for full flow metadata. A short flow with a tiny payload is a beacon. A long flow with megabytes transferred is a real download.
- Check whether the same host touched pastebin.com again in the next 24 hours.
- Confirm whether any other host in the fleet sent a bitsadmin user-agent in the same window.
- Look for lateral movement (4624 type 3 logons, Kerberos TGS) from this host to anywhere else.

## Key Takeaways

- **User-agent is one of the highest-signal fields you have for almost zero cost.** Anomalous user-agents catch a whole class of LOLBin abuse.
- **Trusted services are an attacker's preferred C2.** Domain reputation alone cannot save you. Pair allow-listed destinations with behavior detection.
- **Always preserve the connection identifier.** The Zeek *uid* is the bridge from "one suspicious HTTP record" to "full network flow context."
- **One alert is rarely the whole story.** Confirm parent process, confirm subsequent traffic, confirm whether any other host shows the same pattern.

## What a Defender Should Do

- Alert on every HTTP connection whose user-agent matches a known administrative tool (bitsadmin, certutil, curl, wget, Python-urllib, PowerShell, Go-http-client) coming from a non-administrative workstation.
- Pair user-agent detection with domain-based detection for paste sites, Discord webhooks, Telegram bots, and other commonly-abused C2 covers.
- Make sure your network sensor (Zeek, Suricata, vendor equivalent) emits a flow identifier on every HTTP record. That field is what makes the difference between an alert and an investigation.
- Treat any close-to-the-flag artifact as the start of an investigation, not the end. Document the pivots before closing the ticket.
