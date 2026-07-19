---
title: "Nmap Post Port Scans - Service Detection, OS Detection, and NSE"
date: 2026-07-11
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "What to do after you find open ports: identify the service and version, fingerprint the OS, run NSE scripts, and save the results."
---

A port scan tells you a port is open. This room is about the next step: figuring out what is actually running there, what OS the host is, and pulling extra detail with scripts. It closes out the Nmap series.

## Service and version detection

```bash
nmap -sV TARGET_IP                 # detect the service and version on each open port
nmap -sV --version-light TARGET_IP # fewer probes, faster, less certain
nmap -sV --version-all TARGET_IP   # every probe, slower, more thorough
```

The version string is the point. A service banner like vsftpd 3.0.3 is what you match against known vulnerabilities.

## OS detection and traceroute

```bash
nmap -O TARGET_IP            # fingerprint the operating system
nmap --traceroute TARGET_IP  # map the hops to the target
```

When nmap cannot find an exact OS match it prints the raw fingerprint. You can still read the TTL from it: a T=40 value is hex for 64, which points to Linux. Windows starts at 128.

## Nmap Scripting Engine

NSE scripts live in /usr/share/nmap/scripts and extend nmap well past port scanning.

```bash
nmap -sC TARGET_IP                        # run the default safe scripts
nmap --script=http-robots.txt TARGET_IP   # run a specific script
nmap --script-help SCRIPTNAME             # read what a script does before running it
```

--script-help is worth the habit. It shows the description and category (safe, intrusive, vuln) so you know what a script does before it touches a target. Vuln scripts follow the pattern http-vuln-cveYYYY-NNNN, named after the CVE.

## The shortcut and saving output

```bash
nmap -A TARGET_IP            # -sV, -O, -sC, and --traceroute in one command
nmap -oN out.txt TARGET_IP   # normal output to a file
nmap -oG out.gnmap TARGET_IP # greppable output
nmap -oX out.xml TARGET_IP   # XML output, parseable by other tools
nmap -oA out TARGET_IP       # all three formats at once
```

## Key Takeaways

-sV and -sC are the two you will run most. -A bundles the full workup when you want everything, and -oA saves it in every format so you have a record to work from. That finishes the Nmap module: discovery, basic scans, advanced scans, and now the post-scan detail.