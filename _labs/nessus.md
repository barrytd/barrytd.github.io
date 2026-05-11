---
title: "Nessus"
date: 2026-04-27
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "Configured Nessus Essentials, ran a Basic Network Scan and a Web Application Tests scan, and triaged findings ranging from cleartext login forms to exposed .bak backup configs and missing X-Frame-Options."
---

## Overview

Nessus is an easy TryHackMe room focused on **vulnerability scanner operations**: standing up a scanner, picking the right policy template for the target, triaging the output, and translating raw findings into actionable remediation language.

**Nessus** (by Tenable) is the most widely-deployed enterprise vulnerability scanner. Nessus Essentials is the free tier capped at 16 IPs, and is the standard way new analysts learn the tool. This room is the *"day one on the SOC, here's the scanner, run a scan"* exercise.

## Tools Used

- **Nessus Essentials** for the actual scans.
- **A browser** to view scan results and the host details panel.
- **CVSS** (Common Vulnerability Scoring System, the numeric severity score every CVE carries) for prioritizing findings.
- **MITRE CVE database** to look up specific findings the scanner flags.

## Methodology

**Step 1 - Choose the right scan template.** Nessus ships with dozens of pre-built scan templates. For an unauthenticated external view of a host, the relevant ones are:

- **Basic Network Scan** is the default broad scan: full port range, service version detection, every loaded plugin runs.
- **Web Application Tests** focuses on HTTP-specific checks like missing security headers, deprecated TLS versions, and known web framework vulnerabilities.
- **Advanced Scan** lets you mix policies but is slower and more configuration-heavy.

The right move on a new target is to run the Basic Network Scan first to get the lay of the land, then a Web App Tests pass against any HTTP service that surfaces.

**Step 2 - Configure and run.** A scan in Nessus needs three things: a template, a target list (one or more IPs or CIDR ranges), and optional credentials for an *authenticated* scan. Credentialed scans drastically reduce false positives because the scanner can ask the OS directly *"what version of openssl is installed?"* rather than fingerprinting it from network responses.

**Step 3 - Read the dashboard.** When a scan completes, Nessus presents findings sorted by severity (Critical, High, Medium, Low, Info) with each finding linked to a plugin ID and the relevant CVE. The host details panel shows the OS guess, the open ports, and whether authentication succeeded.

**Step 4 - Triage the findings.** A typical finding row looks like:

```
Plugin 11936    Critical    HTTP - Cleartext Authentication
                            login.php submits credentials over HTTP without TLS.
```

The triage questions for every finding:

1. **Is the affected service actually used?** A vuln in a service no one talks to is lower priority than a vuln in a public-facing service.
2. **Is the finding exploitable in this context?** Some findings are reachable only with local access.
3. **Is there a known public exploit?** Check Exploit-DB or the CVE's references.
4. **What is the remediation?** Usually a patch, a configuration change, or a compensating control.

**Step 5 - Distinguish the categories.** Common buckets the scanner organizes findings into:

- **Missing patches / outdated software** (e.g., openssh older than current LTS). Easy to fix, just update.
- **Configuration weaknesses** (e.g., TLS 1.0 enabled, weak ciphers, missing HSTS). Fixed by editing the service config.
- **Information disclosure** (e.g., a *.bak* backup config readable from /var/www, version banners in headers). Fixed by removing the file or stripping the header.
- **Web app issues** (e.g., missing X-Frame-Options, missing Content-Security-Policy). Fixed by adding the response header.
- **Authentication issues** (e.g., cleartext login forms, default credentials, weak password policy).

**Step 6 - Recognize false positives.** Nessus flags things based on banner fingerprints, which can be wrong. A Debian build of openssh has a version banner that looks older than the patched-back-port actually deployed. *Credentialed scans* eliminate most false positives because they read the actual package version from the OS.

## Key Takeaways

- **A scan is a starting point, not a report.** The output needs triage by a human who understands the environment.
- **Credentialed scans are dramatically more accurate than unauthenticated ones.** If you control the target, run credentialed.
- **CVSS is a prioritization signal, not a verdict.** A critical CVSS finding on an internal host that isn't reachable from anywhere is lower priority than a medium on the internet-facing front door.
- **Information disclosure findings are quiet but load-bearing.** A *.bak* config or a server version banner is often the lever that turns a chain into a critical path.
- **Don't ignore the Info-severity bucket entirely.** It's where banner disclosures and weak ciphers live, and those frequently become foothold material.

## What a Defender Should Do

- Run authenticated Nessus scans on a regular cadence (weekly or biweekly for production, monthly minimum for everything else).
- Patch by *exploitability and reachability*, not by raw CVSS alone. A reachable medium is more urgent than an unreachable critical.
- Strip version banners from HTTP responses, SSH banners, and SMTP greetings. Banners are free fingerprints for every attacker scanning the internet.
- Remove backup files, *.bak* and *.old* and *.tmp*, from any web-served directory. Add a deny rule at the web server config layer for those patterns.
- Add the standard security response headers: *X-Frame-Options*, *X-Content-Type-Options*, *Strict-Transport-Security*, *Content-Security-Policy*, *Referrer-Policy*. They are free hardening.
- Track scan output over time in a ticketing system. The point of scanning is the *trend*: critical findings should be closing, not accumulating.
