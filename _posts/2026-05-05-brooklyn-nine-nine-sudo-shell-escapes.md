---
layout: post
title: "Brooklyn Nine Nine: Two Independent Roads to Root and the GTFOBins Mindset"
date: 2026-05-05
categories: [offensive-security, ctf]
tags: [ftp, ssh, steghide, sudo, gtfobins, methodology]
excerpt: An easy TryHackMe room with a really clean teaching gimmick: two completely separate attack chains land on the same root flag. Both end with sudo on a Unix binary that has a built-in shell escape, which is one of the most generalizable privesc lessons in offensive security.
---

> Companion to my [security-lab-portfolio writeup](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-05-tryhackme-brooklyn-nine-nine). Sanitized: no flag values, passwords, or credentials.

## The Setup

Three open ports: FTP on 21, SSH on 22, HTTP on 80.

![Nmap scan]({{ "/assets/images/brooklyn-nine-nine/01-nmap-scan.png" | relative_url }})

The room offers two independent attack chains, which the writer probably built on purpose so a learner can run both and feel the *"the same lesson, two different shapes"* moment.

## Path 1: FTP, a Hint, Hydra, less

### Anonymous FTP

vsftpd accepts the magic *anonymous* login (any password is fine).

![Anonymous FTP]({{ "/assets/images/brooklyn-nine-nine/02-ftp-anonymous-login.png" | relative_url }})

A *note_to_jake.txt* file sits in the listing. It says, in plain English, *"please change your password, it is too weak."*

![Note to Jake]({{ "/assets/images/brooklyn-nine-nine/03-note-to-jake.png" | relative_url }})

Two pieces of intelligence in one sentence:

1. There is a real account named *jake*.
2. Its password is brute-forceable.

That converts a guess-the-user-and-the-password problem into a guess-just-the-password problem.

### Hydra Against SSH

**Hydra** is a generic credential brute-forcer. Pointed at SSH with a known username and the rockyou wordlist (the famous 14-million-word list shipped with Kali), it returns a hit in seconds.

```
hydra -l jake -P /usr/share/wordlists/rockyou.txt 10.64.148.145 ssh
```

The specific password is intentionally omitted. What matters is that *the password lived near the top of rockyou.txt*, which is the entire reason the room ships with that hint note.

### SSH In, sudo less, Root Shell

A normal SSH login lands a session as jake.

![SSH login]({{ "/assets/images/brooklyn-nine-nine/05-ssh-login.png" | relative_url }})

The next move on every Linux foothold is *sudo -l*, which lists what the current user is allowed to run with sudo. The output names a single binary: *less*.

That is enough. The **less** pager (the program you usually use to scroll through long files) has a built-in shell-escape feature. Inside less, typing `!command` runs that command in a subshell. Because *sudo less* is itself running as root, the subshell inherits root's UID. Typing `!sh` from inside *sudo less* drops you straight into a root shell.

```
sudo less /etc/passwd
!sh
```

## Path 2: HTTP, Steganography, sudo nano

### A Hidden File Inside a Web Image

The Apache server on port 80 hosts an image. **Steganography** is the practice of hiding files inside the imperceptible bits of a media file: the lowest-order bits of a pixel's color or a sample's amplitude, where flipping them produces a change the human eye or ear cannot detect.

**steghide** is the standard CLI tool for embedding and extracting steg-hidden files. The room uses a deliberately weak passphrase (*admin*) so the focus is on the technique, not on guessing.

```
steghide extract -sf <image>.jpg -p admin
```

The extracted file contains a different user's plaintext password. The lesson is **steganography is hiding, not encryption.** Encoding obscures, encryption protects. A weak passphrase reduces steghide to base64 with extra steps.

### sudo nano, Same Bug, Different Binary

Logging into SSH as the other user works. *sudo -l* reveals he can run **/bin/nano** as root, with the **NOPASSWD** flag (sudo will not even prompt for a password).

![Holt sudo -l]({{ "/assets/images/brooklyn-nine-nine/11-holt-ssh-login.png" | relative_url }})

**nano** has the same flavor of feature *less* does: an *Execute Command* function (reached with Ctrl-R then Ctrl-X depending on the build) that prompts for a shell command and runs it in a subshell. Same trick, same privesc, different tool:

```
sudo nano
^R ^X
Command to execute: reset; sh 1>&0 2>&0
```

The *reset* part cleans up nano's terminal state. The *sh 1>&0 2>&0* part reattaches stdout (file descriptor 1) and stderr (descriptor 2) to nano's existing stdin (descriptor 0), so the new shell is fully interactive instead of half-broken.

*whoami* returns **root**, and the same root flag file drops out as in Path 1.

## The Real Lesson: GTFOBins

Both paths end with the same observation: **sudo on a binary with a built-in shell escape is functionally identical to sudo on /bin/sh.** It does not matter whether the binary is *less*, *nano*, *vi*, *vim*, *awk*, *find*, *ftp*, *more*, *perl*, *python*, or any of the dozens of others. They all have documented shell escapes. The community-curated database of them is called **[GTFOBins](https://gtfobins.github.io/)**, and it is the single most valuable bookmark for Linux privilege escalation.

Any *sudo -l* output that names a common Unix utility should be cross-referenced against GTFOBins immediately.

## Vulnerabilities In One Place

- **Anonymous FTP with sensitive content** (CWE-284): disable anonymous (`anonymous_enable=NO`), replace FTP with SFTP for any case where authenticated transfer is needed.
- **Sensitive information disclosure in a public note** (CWE-200): never put account-state hints in any anonymously-readable channel.
- **No brute-force protection on SSH** (CWE-307): install *fail2ban*, set *MaxAuthTries*, require multi-factor.
- **Steganography with a guessable passphrase** (CWE-321 + CWE-1391): never store credentials in steganographic carriers. The carrier file is not the secret; the passphrase is, and it is almost always weak.
- **sudo on a GTFOBins binary** (CWE-250 + CWE-269): remove the sudoers entry, replace with a narrow wrapper script if a real operational need exists, ban NOPASSWD by policy.

## Key Takeaways

- **Anonymous FTP is almost always a *contents* problem, not a *protocol* problem.** The protocol being on is a low finding. What is in the directory is the actual finding.
- **Rockyou plus a known username will land hits in seconds against any password near the top of the list.** The fix is throttling and MFA, not stronger passwords.
- **GTFOBins is the single most valuable bookmark for Linux privesc.** Memorizing the names is fine; checking the site against your sudo -l output every single time is what catches the bug.
- **NOPASSWD on a general-purpose binary is the worst possible sudoers entry.** It removes the password barrier and the audit trail.
- **Steganography is encoding, not encryption.** Encoded data hidden in a public file is functionally cleartext.

The full per-phase walkthrough with screenshots lives on the [portfolio repo](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-05-tryhackme-brooklyn-nine-nine).
