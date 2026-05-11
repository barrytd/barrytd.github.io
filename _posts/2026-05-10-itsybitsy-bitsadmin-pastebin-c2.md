---
layout: post
title: "Hunting a bitsadmin Beacon to Pastebin in Kibana"
date: 2026-05-10
categories: [blue-team, detection]
tags: [kibana, elk, lolbin, mitre-attack, t1197, t1105, soc]
excerpt: A walkthrough of TryHackMe's ItsyBitsy room. An IDS alert on a workstation in HR turns into a hunt for an unusual user-agent string in Kibana, and the trail ends on a Pastebin paste being used as a tiny command-and-control channel.
---

> Companion piece to my [security-lab-portfolio writeup](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-10-tryhackme-itsybitsy). Sanitized for the public blog: no flag values, just the technique.

## The Scenario

Imagine you are sitting on a SOC console and an IDS rule fires for a host in the **HR** department. The alert is vague: *"potential C2 communication"*. You have an Elasticsearch index of HTTP connection logs and a short window of time in March 2022 to look at. Your job is to figure out whether it's real, what happened, and what to recommend.

This is exactly the loop ItsyBitsy puts you in, and it is a really clean introduction to **detection engineering** (the discipline of writing the rules and queries that catch attackers, as opposed to just responding to alarms after the fact).

## The Tools

- **ELK Stack** stands for *Elasticsearch + Logstash + Kibana*. Elasticsearch stores the logs, Logstash ingests them, and Kibana is the web UI you actually use day to day. In this room you only touch Kibana.
- **Kibana Discover** is the *"search every log line"* view. You point it at an index, set a time range, and start filtering.
- **An index** in Elasticsearch is the equivalent of a database table. This room ships with one called *connection_logs*.

## Scope the Window First

The very first thing to do with any timeline-driven investigation is *narrow the time window*. The alert names March 2022, so I set Kibana's time range to Mar 1, 2022 to Mar 31, 2022.

![March 2022 events]({{ "/assets/images/itsybitsy/01-march-2022-events.png" | relative_url }})

The histogram at the top is the analyst's friend. Even at a glance you can see a spike that does not belong to the baseline noise of normal HTTP traffic. That spike is the part of the timeline worth zooming into first.

**1,482 events** across the month is a small enough volume to scan field by field instead of building heavy aggregations. In a real environment you would likely have hundreds of thousands of records and have to pivot through aggregations first. The technique is the same; just the scale changes.

## What to Look At First

When you do not know what you are looking for, the smartest first move is to look at *fields with low natural variety* in the data. HTTP traffic in a healthy environment has surprisingly few distinct **user-agent** strings: Chrome, Firefox, Edge, maybe an Office update agent. Anything else is unusual.

I scrolled through the per-event view and found one record whose **user_agent** field read *bitsadmin*. That is not a browser. That is the name of a Windows command-line tool, which means a human or a script ran something on the box rather than a person clicking a link.

![Suspicious record - upper half]({{ "/assets/images/itsybitsy/02-key-log-entry.png" | relative_url }})

![Suspicious record - lower half]({{ "/assets/images/itsybitsy/03-key-log-entry-2.png" | relative_url }})

## What bitsadmin Is and Why It Matters

**BITS** stands for *Background Intelligent Transfer Service*. It is a built-in Windows feature that lets the operating system download files in the background, slowly, without saturating the network. Windows Update uses it. So does SCCM and a lot of other Microsoft tooling.

**bitsadmin.exe** is the command-line front end to BITS. It is signed by Microsoft and ships with every Windows install, which makes it a textbook **LOLBin** (*"living off the land binary"*, security-speak for a trusted built-in tool that attackers reuse to do something it was not really designed for).

Mapped to **MITRE ATT&CK** (the public framework for naming attacker behaviors so defenders can talk about them consistently):

- **T1197 - BITS Jobs** is the exact technique of using bitsadmin to fetch a payload.
- **T1105 - Ingress Tool Transfer** is the umbrella for "malware retrieves more malware from the internet."
- **T1071.001 - Application Layer Protocol: Web Protocols** covers the choice of plain HTTP to a popular site rather than a custom protocol.

## What Pastebin Has to Do With It

The same record showed the host connecting outbound to **pastebin.com**. Pastebin is a legitimate service where anyone can host a snippet of text under a short opaque URL. Real developers use it daily, which means most corporate proxies allow it.

That is also exactly why attackers love it. A small HEAD or GET to a Pastebin URL on a schedule is indistinguishable from a developer checking a colleague's paste. The payload is hidden inside an allowed destination, and TLS hides the contents from most network DLP tools.

## A Tiny Detection Rule

In Kibana's query bar, the literal Lucene-style query that would catch *just this event* in real time is:

```
user_agent : "bitsadmin" AND host : "pastebin.com"
```

A more general production version would broaden both axes:

```
user_agent : ("bitsadmin" OR "curl" OR "wget" OR "python-requests" OR /PowerShell.*/) 
  AND host : (/pastebin\.com/ OR /paste\.ee/ OR /ghostbin\.com/ OR /hastebin\.com/)
```

The list of paste sites is the brittle part because attackers rotate constantly. The user-agent allow-list is the part that ages well.

## What a Real SOC Would Do Next

The room ends when you find the Pastebin and read what it says. A real investigation would treat that read as the **start** of the case, not the end. Concretely:

1. **Endpoint pivot.** Ask EDR for the *parent process* of bitsadmin.exe on the workstation at the timestamp from the log. bitsadmin launched by *winword.exe* or by a script host is way more interesting than bitsadmin launched by an admin in cmd.exe.
2. **Network pivot.** Correlate the **Zeek uid** (the connection identifier in the log) back to the parent conn.log entry for full flow metadata. A short flow with a tiny payload is a beacon. A long flow with megabytes transferred is a real download.
3. **Subsequent traffic.** Did this host touch pastebin.com again in the 24 hours after the alert? Did any other host in the fleet send a bitsadmin user-agent in the same window? Campaigns rarely hit one host.
4. **Lateral movement.** The HR workstation is a beachhead, not a target. Look for Kerberos and 4624 type-3 logons from this host to anywhere else.

## Key Takeaways

- **User-agent is one of the highest-signal fields you have for almost zero cost.** Anomalous user-agents catch a whole class of LOLBin abuse.
- **Trusted services are an attacker's preferred C2.** Domain reputation alone cannot save you. Pair allow-listed destinations with behavior detection.
- **Always preserve the connection identifier.** The Zeek *uid* is the bridge from "one suspicious HTTP record" to "full network flow context." It is the difference between an alert and an investigation.
- **One alert is rarely the whole story.** Confirm parent process, confirm subsequent traffic, confirm whether any other host in the environment shows the same pattern.

The phase-by-phase walkthrough with screenshots lives on the [companion writeup on GitHub](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-10-tryhackme-itsybitsy).
