---
title: "Vulnerability Scanning with Nmap and Nessus on Metasploitable 2"
date: 2026-09-27
platform: Self-built
difficulty: Easy
category: blue-team
summary: "A weekend lab running the full vulnerability management cycle against Metasploitable 2. Nmap for discovery, Nessus for scanning, then triage and a report. Non-credentialed found 61 issues. Credentialed found 91."
---

## What This Lab Covers

Most of my labs so far have been offensive. This one is the other side of the desk: the vulnerability management workflow an analyst runs. Discover what is on the network, scan it, decide what matters, hand it off.

The target is Metasploitable 2, an intentionally vulnerable Linux box. Everything runs on a VirtualBox host-only network, so the target can reach my Kali box and my PC but not the internet or my home network. That isolation matters when the machine you are scanning is full of holes.

## Step 1: Discovery with Nmap

Before scanning for vulnerabilities you map what is there. This is the asset inventory step.

```bash
nmap -sV 192.168.56.103
```

`-sV` pulls service versions, so it does not just say port 21 is open, it says `vsftpd 2.3.4`. Versions are what you match against known CVEs later.

![Nmap service and version scan]({{ "/assets/images/nessus-metasploitable/nmap-service-scan.png" | relative_url }})

23 open ports. Old software everywhere, plus a leftover backdoor on port 1524. A normal server exposes three or four ports. Every one of these is attack surface.

OS detection needs raw sockets, so it runs with sudo:

```bash
sudo nmap -O 192.168.56.103
```

Result: Linux 2.6, Ubuntu 8.04. That dates the box to around 2008.

## Step 2: Non-Credentialed Nessus Scan

In Nessus I ran a Basic Network Scan against the target with no login. This is the outside-in view, what an attacker on the network sees.

![Nessus non-credentialed results]({{ "/assets/images/nessus-metasploitable/nessus-noncred-results.png" | relative_url }})

61 findings: 4 critical, 3 high, plus mediums and lows.

## Step 3: Triage

Three scores drive the order of work:

- **CVSS**: how severe the flaw is, 0 to 10.
- **EPSS**: the chance it gets exploited soon.
- **VPR**: Tenable's blended score using observed threat activity.

The rule is to fix first what is both severe and easy to exploit, not just the highest number.

![VNC 'password' password finding]({{ "/assets/images/nessus-metasploitable/vnc-finding.png" | relative_url }})

The clearest example was a VNC remote desktop service with the password set to the word "password". CVSS 10, remote, no skill needed. That is the real fix-today item.

Then the lesson that stuck. Samba Badlock scored 7.5 with a 37 percent EPSS, which looked urgent. But the finding detail said no known exploit and unproven code maturity, and the CVSS vector had high attack complexity and required user interaction. So the attacker would need to already be sitting in the middle of the traffic. I ranked it below the easy wins.

EPSS is a prediction. The finding detail is evidence. When they disagree, read the finding.

## Step 4: Credentialed Scan

Then I ran the same scan again with SSH credentials so Nessus could log into the host.

![Credentialed scan results]({{ "/assets/images/nessus-metasploitable/credentialed-results.png" | relative_url }})

The count went from 61 to 91. With a login, Nessus read the installed packages and patch levels and found things the outside scan could not see:

- Bash Remote Code Execution (Shellshock), CVSS 9.8, EPSS 1.0. A known, actively exploited flaw.
- Weak Debian OpenSSH Keys, CVSS 9.8.
- Canonical Ubuntu Linux missing patches, 229 grouped items.

A non-credentialed scan shows what an attacker sees from the network. A credentialed scan shows what is actually installed and unpatched inside the host. Run credentialed when you have access. It is more complete with fewer false positives.

## What I Would Do On The Job

Open a ticket per finding for the asset owner, whether that is IT or the product owner, so they can prioritize and assign the fix. Give management a short summary with the risk level, not the raw scan.

## Takeaways

- Host-only networking keeps a vulnerable box safe to study.
- Nmap maps the attack surface. Nessus scores it.
- CVSS is the start, not the end. EPSS and the finding detail can change the order.
- An end-of-life OS cannot be patched. The fix is replacement or isolation.
