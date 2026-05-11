---
title: "Overpass 2 - Hacked"
date: 2026-04-23
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "PCAP forensics to recover a su password, cracked a deployed SSH backdoor's hash in hashcat, and exploited an .suid_bash with -p to root."
---

## Overview

Overpass 2 is the *defensive forensics* sequel to Overpass: a previous attacker has already compromised the box and deployed a backdoor, and the job is to figure out what they did from network captures and a hash file, then walk in through the same back door they did.

This room teaches **PCAP forensics**, **offline hash cracking**, and the **SUID bash -p** privesc pattern, all useful core skills.

## Tools Used

- **nmap** for port scanning.
- **Wireshark** for the PCAP analysis.
- **strings** and **grep** for sifting through PCAP exports.
- **hashcat** for offline cracking of the recovered backdoor hash.
- **ssh** for the foothold.
- **find** for SUID enumeration.

## Methodology

**Step 1 - Get the PCAP.** The room ships with a packet capture of the previous attacker's session. Opening it in Wireshark and following the TCP streams (Right-click any packet → *Follow → TCP Stream*) reproduces the attacker's terminal session as plaintext.

**Step 2 - Read what the attacker did.** The follow-stream output shows the attacker downloading an exploit, popping a shell, *su*-ing to a privileged account, and *deploying a backdoor script*. Three valuable pieces fall out:

- The cleartext password they typed during *su* (because telnet and unencrypted SSH session content shows up plain in PCAP).
- The full script of the deployed backdoor (a custom hashed SSH backdoor).
- The hash format the backdoor uses to authenticate.

**Step 3 - Read the backdoor script.** The backdoor is a small piece of Python that accepts SSH connections on a non-standard port and lets in any user whose typed password salts and hashes to a hardcoded value. The hash algorithm (a *specific* hashcat mode) is visible right in the script.

**Step 4 - Crack the backdoor hash.** hashcat in the matching mode crunches the rockyou wordlist against the hardcoded hash.

```bash
hashcat -m <MODE> -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

`-m` is the hash mode (algorithm). `-a 0` is straight dictionary attack. The hit returns in seconds because the password lives near the top of rockyou.

**Step 5 - Walk in through the back door.** SSH to the box with the cracked credentials lands a shell.

**Step 6 - SUID bash privesc.** Enumerating SUID files surfaces a hidden binary in the user's home: */home/james/.suid_bash*.

```bash
find / -perm -4000 -type f 2>/dev/null | grep -v Permission
```

This is **SUID bash**. By default, bash drops privileges when started from a SUID binary (this is the bash *POSIX mode* safety check). The **-p** flag tells it not to, which is why any SUID-bit bash launched with -p is effectively sudo without a password.

```bash
./.suid_bash -p
# .suid_bash-4.4#
```

## Key Takeaways

- **Unencrypted protocols leak everything in PCAP.** Telnet, FTP, plain HTTP, even some misconfigured SSH key exchanges all reveal credentials in plaintext.
- **Backdoor scripts are often readable.** When you find one, read the source. The hash algorithm, the trigger, and the credential format are all in there.
- **hashcat with rockyou is the fastest first attempt** for any unknown hash that does not look obviously expensive (Argon2, bcrypt with high cost factor).
- **SUID bash -p is the canonical *"how did this happen"* privesc.** It's almost always there because an admin tried to make their own *"sudo without password"* shortcut.

## What a Defender Should Do

- Disable plaintext protocols at the network and host level: no Telnet, no FTP, no plain HTTP-authenticated services. Force TLS or SSH.
- Monitor for new SUID binaries with file integrity tooling. A new file with the SUID bit appearing in a user home directory is a high-confidence backdoor indicator.
- Strip writable permissions from `/etc/ssh/sshd_config.d/` and `/etc/ssh/sshd_config` for non-root accounts. Backdoors often add a *Match User* block.
- Rotate every credential that has ever been typed over an unencrypted session, especially after a confirmed compromise.
- Keep PCAP capture running on critical segments. The forensics value of PCAP is enormous and storage is cheap.
