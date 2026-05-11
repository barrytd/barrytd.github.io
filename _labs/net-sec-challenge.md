---
title: "Net Sec Challenge"
date: 2026-04-15
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "Network Security module capstone covering Nmap, Telnet banner grabbing, and Hydra against six services on a single target, plus a 0%-detection covert scan."
---

## Overview

Net Sec Challenge is the capstone exercise for TryHackMe's Network Security module. The room tests a half dozen networking skills against one host: **full-range port scanning**, **banner grabbing with Telnet**, **Hydra against non-standard ports**, and **covert nmap scanning** to evade an IDS challenge.

The lesson cluster: the default nmap *top-1000* scan misses services hiding on non-standard ports, server banners leak version information by default, and aggressive scanning is *trivially detected* in any monitored environment.

## Tools Used

- **nmap** for port discovery and version detection (multiple invocations).
- **Telnet** for banner grabbing against the SSH and HTTP services.
- **hydra** for FTP brute force on a non-standard port.
- **A browser** for the covert-scan tracking challenge.

## Methodology

**Step 1 - Full TCP port scan.** The default nmap scan only hits the top 1000 ports. A real engagement needs every port.

```bash
nmap -p- -sV <TARGET_IP>
```

`-p-` is shorthand for *all 65535 TCP ports*. Six open ports surface, including one on **10021** running an unknown service. The default scan would have missed it entirely.

**Step 2 - HTTP banner inspection.** Many services embed metadata in their version strings. Visiting the web server on port 80 in a browser and inspecting the response headers (DevTools → Network → click any response → Headers) shows the *Server* header containing version-revealing text. The Server header in HTTP responses is one of the cheapest free fingerprints any attacker gets.

**Step 3 - SSH banner grab via Telnet.** Telnet is not just a remote shell. It is also a generic *raw TCP connector*, which means it can be used to grab the version banner of any TCP service that emits one on connect.

```bash
telnet <TARGET_IP> 22
# returns something like:
# SSH-2.0-OpenSSH_8.2p1 <version_disclosure>
```

The SSH version string is the part to remember. Banner grabbing without authentication is the cheapest reconnaissance step there is.

**Step 4 - Fingerprint the unknown service on 10021.** A targeted version scan against the unusual port confirms it as **vsftpd 3.0.5** on a non-standard FTP port.

```bash
nmap -p10021 -sV <TARGET_IP>
```

**Step 5 - Hydra against non-standard FTP.** Hydra's default for FTP is port 21. The `-s` flag overrides the port.

```bash
hydra -l <username> -P /usr/share/wordlists/rockyou.txt <TARGET_IP> ftp -s 10021
```

`-l` is a known username (recovered through context). `-P` is the wordlist. `-s` is the *service port*. Without `-s`, Hydra would have happily run for hours against the closed port 21 and reported no hits.

**Step 6 - Covert nmap scan.** The room's final challenge is a browser-tracked scan that measures how *quietly* the scan was performed. Aggressive nmap defaults light up every IDS in the building.

The techniques that lower the detection percentage:

- **SYN scan (`-sS`):** half-open connections that don't complete the three-way handshake. Some old IDS rules only log full connects.
- **Timing controls (`-T0` to `-T2`):** *Paranoid*, *Sneaky*, *Polite* slow the scan to a crawl. Each step lowers the packets-per-second rate.
- **Fragmentation (`-f`):** splits packets into smaller pieces that may slip past pattern matching.
- **Decoy IPs (`-D RND:10`):** sprays the same packets from multiple spoofed source IPs so the real source is buried.

```bash
nmap -sS -T2 -f -D RND:5 <TARGET_IP>
```

Achieving 0% detection in the room is a function of *combining* these flags. None of them alone gets there.

## Key Takeaways

- **Port range matters.** The default nmap top-1000 scan misses services on non-standard ports. Always run `-p-` on a real target.
- **Server banners are free reconnaissance.** Strip them in production. They are also the cheapest detection signal for the defender.
- **Telnet is a legitimate banner-grabbing tool.** Connecting to any TCP port with Telnet returns whatever the service sends before authentication.
- **Hydra against non-standard ports needs `-s`.** Forgetting it is a classic beginner mistake.
- **Covert scanning is a real operational constraint.** IDS evasion techniques exist because aggressive scanning is trivially detected.

## What a Defender Should Do

- Block scans at the network edge with rate-limited ICMP and TCP responses. Snort/Suricata rules for nmap signatures are mature and free.
- Strip version banners from every service: *Server* header in HTTP, SSH version string, SMTP greeting. Set `ServerTokens Prod` in Apache, `server_tokens off` in nginx.
- Move administrative services off their default ports as a *defense in depth* measure (it raises the cost of casual scans, but real scans like `-p-` find them anyway).
- Rate-limit authentication on every login endpoint. Hydra is only effective when the target accepts unlimited attempts.
- Monitor for slow port scans, not just fast ones. A real attacker uses `-T0` to evade detection; your rules should catch that pattern with a long aggregation window.
