---
title: "Brooklyn Nine Nine"
date: 2026-05-05
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "Two completely independent attack chains land on the same root: anonymous FTP plus rockyou Hydra plus sudo less escape, and steghide on a web image plus sudo nano with NOPASSWD."
---

## Overview

Brooklyn Nine Nine is an easy TryHackMe room with a clean teaching gimmick: **two completely independent attack chains** land on the same root flag. The writer almost certainly built it that way so a learner can run both and feel the *"the same lesson, two different shapes"* moment.

Both paths end on the same class of bug: **sudo on a Unix binary that has a built-in shell escape**. That bug is one of the most generalizable Linux privesc patterns there is, and the community-curated reference for it (GTFOBins) is the single most useful bookmark in offensive security.

## Tools Used

- **nmap** for port scanning.
- **ftp** for the anonymous foothold.
- **hydra** with the rockyou wordlist for SSH brute force.
- **ssh** for the foothold and pivot.
- **steghide** for the steganographic extraction.
- **less** and **nano** as the privesc primitives.
- **GTFOBins** as the mental reference for *"which binaries can I shell-escape from under sudo."*

## Methodology

### Path 1: FTP, a Hint, Hydra, less

**Step 1 - Anonymous FTP.** vsftpd accepts the magic *anonymous* login (any password is fine).

```bash
ftp <TARGET_IP>
# Name: anonymous
# Password: (anything)
```

A *note_to_jake.txt* file sits in the listing. It says, in plain English, *"please change your password, it is too weak."* Two pieces of intelligence in one sentence: there is a real account named *jake*, and his password is brute-forceable. That converts a *guess the user AND the password* problem into a *guess just the password* problem.

**Step 2 - Hydra against SSH.** **Hydra** is a generic credential brute-forcer. Pointed at SSH with a known username and the rockyou wordlist (the famous 14-million-word list shipped with Kali), it returns a hit in seconds.

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt <TARGET_IP> ssh
```

`-l` is a single username. `-P` is the password wordlist. The specific password is intentionally omitted; what matters is that the password lived near the top of rockyou, which is the entire reason the room ships with the hint note.

**Step 3 - sudo less to root.** A normal SSH login lands a session as jake. The first move on every Linux foothold is *sudo -l*, which lists what the current user is allowed to run with sudo.

```bash
sudo -l
```

The output names a single binary: *less*. That is enough. The **less** pager (the program you usually use to scroll through long files) has a built-in *shell-escape* feature. Inside less, typing `!command` runs that command in a subshell. Because *sudo less* is itself running as root, the subshell inherits root's UID.

```bash
sudo less /etc/passwd
# inside less, type:
!sh
```

A root shell.

### Path 2: HTTP, Steganography, sudo nano

**Step 4 - Steganography from a public image.** The Apache server on port 80 hosts an image. **Steganography** is the practice of hiding files inside the imperceptible bits of a media file: the lowest-order bits of a pixel's color or a sample's amplitude, where flipping them produces a change the human eye or ear cannot detect. **steghide** is the standard CLI tool for embedding and extracting steg-hidden files.

```bash
steghide extract -sf <image>.jpg -p admin
```

`-sf` is the source file. `-p` is the passphrase (the room uses a deliberately weak one to keep the focus on the technique). The extracted file contains a different user's plaintext password.

**Step 5 - sudo nano with NOPASSWD.** Logging into SSH as the second user works. *sudo -l* this time reveals that the user may run **/bin/nano** as root with the **NOPASSWD** flag, meaning sudo will not even prompt for a password.

```bash
holt@brookly_nine_nine:~$ sudo -l
User holt may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /bin/nano
```

**nano** has the same flavor of feature *less* does: an *Execute Command* function (reached with Ctrl-R then Ctrl-X) that prompts for a shell command and runs it in a subshell. Because nano is running under sudo, the subshell starts as root. The payload below cleans up nano's terminal state and reattaches file descriptors so the new shell is fully interactive.

```bash
sudo nano
# inside nano:
^R ^X
Command to execute: reset; sh 1>&0 2>&0
```

*whoami* returns **root**, and the same root flag file drops out.

## Key Takeaways

- **Anonymous FTP is almost always a contents problem, not a protocol problem.** The protocol being on is a low finding. What is in the directory is the actual finding.
- **Rockyou plus a known username will land hits in seconds against any password near the top of the list.** The fix is throttling and MFA, not stronger passwords.
- **GTFOBins is the single most valuable bookmark for Linux privesc.** Memorize the names, check every *sudo -l* output against the site.
- **NOPASSWD on a general-purpose binary is the worst possible sudoers entry.** It removes the password barrier and the audit trail.
- **Steganography is encoding, not encryption.** Encoded data hidden in a public file is functionally cleartext when the passphrase is weak.

## What a Defender Should Do

- Disable anonymous FTP (`anonymous_enable=NO` in /etc/vsftpd.conf). If file sharing is needed, replace FTP with SFTP, which rides inside SSH.
- Never put account-state hints in any anonymously-readable channel. The "please change your password" note was the entire foothold.
- Install *fail2ban* and set sshd's *MaxAuthTries* and *LoginGraceTime* knobs to block brute force at the host level. Require multi-factor for SSH.
- Audit the sudoers file regularly. Remove any entry that points at a GTFOBins binary unless the operational need is real and documented. If a user genuinely needs to read root-owned files, give them a narrow wrapper script with NOPASSWD, not a general-purpose tool with a shell escape.
- Never store credentials in steganographic carriers. The carrier file is not the secret; the passphrase is, and it is almost always weak.
