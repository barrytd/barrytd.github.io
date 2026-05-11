---
title: "SOC Simulator - Phishing Analysis"
date: 2026-03-25
platform: TryHackMe
difficulty: Medium
category: blue-team
summary: "Five phishing alerts triaged in Splunk: two false positives, three true positives. Pivoted between email logs, URL reputation, and firewall data for each verdict."
---

## Overview

SOC Simulator is a TryHackMe practice room that mimics the rhythm of a real SOC shift: alerts come in, the analyst opens each in Splunk, walks the data, and decides *true positive* (real attack) or *false positive* (benign). Five phishing alerts in one session.

The lesson is the **decision tree**: each alert type has a small number of pivots that resolve the question. Once you know the pivots, the work is fast.

## Tools Used

- **Splunk** for SPL (Search Processing Language) queries and dashboards.
- **VirusTotal** and **URLScan** for URL and attachment reputation.
- **Whois / DNS** for sender domain reputation.
- **Email logs** (header analysis) for the actual *did this email arrive* check.
- **Firewall logs** for *did anyone click*.

## Methodology

### The Phishing Triage Decision Tree

For every phishing alert, the same five-step loop:

1. **Confirm the email actually arrived.** Search the email gateway logs for the alerted message-ID. If the gateway blocked it, the alert is a *false positive* (the rule fired but the threat never reached the user).
2. **Inspect the headers.** Look at *Received* chain, *Return-Path*, *Reply-To*, SPF/DKIM/DMARC results. Inconsistencies are the strongest single signal.
3. **Inspect the body.** Look at the visible link target vs the actual href. Look at the displayed sender vs the From header. Check for urgency phrasing (*"your account will be locked"*).
4. **Inspect the attachment, if any.** Hash the file, check VirusTotal. Open in a sandbox if necessary.
5. **Check for click-through.** Search firewall and proxy logs for any user connecting to the bad URL. If yes, the case becomes an incident; if no, the case is *contained*.

### Walkthrough of Two Verdicts

**Alert 1 - True Positive:** An email from *paypa1.com* (note the digit-1) to several finance team members with a *"verify your account"* link to a credential harvesting page. SPF was *softfail*, DKIM was missing, the link target on the firewall logs showed two users clicked. **True positive, escalate to IR for the two clickers.**

**Alert 2 - False Positive:** An email containing the word *"invoice"* with an attached PDF, flagged by the rule as potential payload delivery. Header check: legitimate vendor, SPF pass, DKIM pass, DMARC pass. Body check: matches the user's normal invoice format for the vendor. Attachment hash: matches a known legitimate PDF from the vendor (no VT hits). **False positive, close.**

### SPL Queries Used

The two queries that come up constantly in phishing triage:

```spl
| `email_logs` 
| search subject="*invoice*" OR subject="*verify*" 
| stats count by sender, recipient, subject
```

This identifies the messages that match the alert criteria over a time range.

```spl
| `proxy_logs` 
| search url="*paypa1.com*"
| stats count by src_ip, user, url
```

This shows *who clicked the bad URL* and *from where*, which is the difference between *alert* and *incident*.

## Key Takeaways

- **Phishing triage is a decision tree.** Once you know the pivots (header chain, SPF/DKIM/DMARC, attachment hash, click-through), each alert resolves in minutes.
- **The most important pivot is "did the email actually arrive?"** Gateway-blocked alerts are extremely common and should close as false positive immediately.
- **Click-through data converts an alert into an incident.** No clicks means contain-and-close. Clicks mean credentials are gone and accounts need to be reset.
- **VirusTotal and URLScan are free, fast, and accurate enough for first-pass triage.** A bad reputation hit is high-confidence; a clean result with low submission count is *I don't know yet*.

## What a Defender Should Do

- Enforce SPF, DKIM, and DMARC on the org's own email domain. Without DMARC, the org is a sitting duck for spoofing.
- Tune the email gateway to block *display name spoofing* (mismatch between From display name and actual From header) and *lookalike domains* (paypa1.com, micros0ft.com).
- Train users to report phishing. The user report is itself an alert source, often faster than automation.
- Configure browser-level URL reputation (Microsoft SmartScreen, Google Safe Browsing) to interstitial warn on known phishing domains.
- Build automated playbooks that, on a confirmed phish, lock the affected user accounts, reset passwords, search for inbox forwarding rules, and check sent-mail logs for downstream phishing.
- Track *time-to-detect* and *time-to-contain* per phish. Both should be measured in single-digit minutes for a mature SOC.
