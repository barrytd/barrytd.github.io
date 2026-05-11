---
layout: post
title: "Skynet: Five Findings, One Attack Chain, From Anonymous SMB to Root"
date: 2026-05-10
categories: [offensive-security, ctf]
tags: [smb, squirrelmail, cuppa-cms, rfi, tar-wildcard, methodology]
excerpt: A medium-difficulty TryHackMe room that stitches five different low-severity findings into a single compromise. Each step alone looks small. Together they walk you from "no credentials" to "root" through a webmail inbox, an obscured URL, and an unmaintained CMS.
---

> Companion to my [security-lab-portfolio writeup](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-10-tryhackme-skynet). Sanitized: no flag values, passwords, or recovered credentials.

## Why This Room Is Worth Doing

Real attacks rarely look like *"found one critical CVE, popped a shell."* They look like *"found five small things, chained them."* Skynet is one of the cleanest hands-on examples of that pattern I have done. Every individual finding looks like a low-severity hygiene issue. The total is a full root compromise. That gap, between *individual severity* and *chain severity*, is the whole skill.

## What's Open

A standard port scan with **nmap** reveals six services:

![Nmap scan]({{ "/assets/images/skynet/01-nmap-scan.png" | relative_url }})

- SSH on 22 (for if you find SSH-able credentials)
- HTTP on 80 (a web app and SquirrelMail)
- POP3 on 110 and IMAP on 143 (the back end of that webmail)
- Samba on 139 and 445 (Windows-style file shares)

The hostname Samba advertises is **SKYNET** in the workgroup **WORKGROUP**, so this is a Linux box impersonating Windows-style SMB.

## Step 1: Enumerate the Web Root

A **gobuster** directory brute force surfaces the predictable WordPress-ish directories plus a *squirrelmail* path. SquirrelMail is an old-school PHP webmail front end that talks to the POP3/IMAP services on the same host.

![Gobuster directories]({{ "/assets/images/skynet/02-gobuster-dir.png" | relative_url }})

The login page tells me right at the top: *SquirrelMail version 1.4.23 [SVN]*.

![SquirrelMail login]({{ "/assets/images/skynet/03-squirrel-mail-login-screen.png" | relative_url }})

Version banners on login pages are an unintentional gift to attackers. They turn *"figure out what this is"* into *"look up known CVEs for this exact version."*

## Step 2: Find a Real Username

You cannot brute force a login if you do not know the username. **enum4linux** is the swiss-army knife for asking a Samba server *"who lives here?"* It uses anonymous SMB sessions to enumerate user SIDs (the security identifiers Windows assigns to every account) and resolves them back to usernames.

![enum4linux user]({{ "/assets/images/skynet/04-enum4linux-user.png" | relative_url }})

This is **SID enumeration**, and it is the single most useful unauthenticated technique against Samba and Windows file shares. It converts a two-axis brute force *(guess the user AND the password)* into a one-axis brute force *(just guess the password)*.

## Step 3: Look Through the Anonymous Share

The Samba server allows fully anonymous access to a share. Listing it reveals:

![Anonymous SMB]({{ "/assets/images/skynet/05-smb-anonymous-login.png" | relative_url }})

- An *attention.txt* note
- A *logs* directory

The note is a fake operator memo that tells the user (the one whose name I just enumerated) to change his password because *"various passwords were changed after a system malfunction."*

![attention.txt]({{ "/assets/images/skynet/06-attention-txt-file.png" | relative_url }})

This is **information disclosure** (CWE-200, in the official CWE catalog of weakness types). A defender would call it "a memo on a public share" and dismiss it. An attacker reads it as *"there is a user named milesdyson and his password rotation history is probably nearby."*

## Step 4: The Wordlist *Was* the Loot

Inside the logs directory is *log1.txt*. It looks like a log, but it is really a password wordlist of every variant the user has ever picked.

![Password wordlist]({{ "/assets/images/skynet/07-log-wordlist.png" | relative_url }})

This is the part that always teaches me something new: **the file did not need to look like a credential dump to be one.** A user's password history, even if every entry is expired, leaks their *password-generation habits*, which seeds a much smaller brute-force list than rockyou.

## Step 5: Hydra Against SquirrelMail

SquirrelMail's login form takes username and password as POST parameters. **Hydra** is told which fields are which, which wordlist to try, and what the failure-text marker looks like (the literal *"Unknown user or password incorrect"* phrase that appears in a failed-login page). When Hydra sees a response that does *not* contain that phrase, it knows it found a hit.

I let Hydra rip with `-l milesdyson` and `-P log1.txt`. Within seconds it returns a successful login. The exact password is intentionally left out of this post; what matters is the technique.

## Step 6: The Inbox Hands Over Another Password

Logging into SquirrelMail with the recovered credentials reveals three emails. One is from *skynet@skynet* with the subject **"Samba Password reset"**.

![SquirrelMail inbox]({{ "/assets/images/skynet/09-squirrel-login.png" | relative_url }})

It contains the new password for milesdyson's *personal* SMB share, which is different from the anonymous one.

This is the **"compromise one mailbox, compromise every account whose reset workflow lands there"** pattern. It is one of the most common credential-leak vectors in real engagements, and it is the reason mature companies refuse to send credentials by email at all.

## Step 7: Personal Share, Hidden URL

The personal share contains academic PDFs (Skynet flavor: papers on neural networks) and a *notes/* directory. Inside notes is a to-do list that includes the URL of a *beta CMS* the user is working on, sitting at a deliberately random path:

![Hidden directory hint]({{ "/assets/images/skynet/12-smb-hidden-directory.png" | relative_url }})

This is **security through obscurity** (CWE-1392), which means relying on a secret URL or filename instead of authentication. It holds for exactly as long as nobody writes the path down anywhere readable. Spoiler: somebody always does.

Browsing the URL loads a real web app.

![Personal page]({{ "/assets/images/skynet/13-hidden-directory-page.png" | relative_url }})

## Step 8: Cuppa CMS and Its Documented RCE

A second gobuster pass inside the hidden directory turns up */administrator*.

![Hidden gobuster]({{ "/assets/images/skynet/14-gobuster-hidden-directory.png" | relative_url }})

The admin panel belongs to **Cuppa CMS**.

![Cuppa CMS admin]({{ "/assets/images/skynet/15-cms-admin-login.png" | relative_url }})

**searchsploit** is the offline mirror of the Exploit-DB database that ships with Kali. Asking it about Cuppa CMS returns a single hit, **CVE-2018-16763**, with a working public exploit.

![searchsploit]({{ "/assets/images/skynet/16-cuppa-cms-searchsploit.png" | relative_url }})

The bug is a textbook **Remote File Inclusion** (RFI): a query parameter named *urlConfig* is concatenated directly into PHP's *include()* function with no allow-list. PHP's *allow_url_include* must be on for this to work, and it was. So *"give me a URL"* turns into *"the server will fetch that URL and execute whatever PHP it contains."*

This is unauthenticated RCE. I host a copy of the **pentestmonkey** PHP reverse shell on a local Python HTTP server, point the *urlConfig* parameter at it, and a netcat listener catches a callback as **www-data**.

![Reverse shell]({{ "/assets/images/skynet/17-reverse-shell.png" | relative_url }})

## Step 9: Privesc Through a tar Wildcard Injection

Enumerating cron and writable paths shows that root runs a backup script every minute. The script does roughly:

```bash
cd /var/www/html && tar -czf /var/backups/backup.tgz *
```

The *\** (a shell **wildcard** or glob) is what tar receives as arguments. The shell expands it into the literal list of filenames in the directory. Here is the catch: **tar interprets arguments that start with `--` as command-line flags, not as filenames.**

I create two empty files in *that exact directory* whose *names* are valid tar options:

```bash
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

Plus a real script *shell.sh* with a reverse-shell payload. When cron fires next, the shell expands the asterisk into *"--checkpoint=1 --checkpoint-action=exec=sh shell.sh other_files..."*, which tar dutifully treats as *"run shell.sh at each checkpoint."* It runs *as root.*

This is the **tar wildcard injection**, and it generalizes to any cron'd tool that accepts wildcards (rsync, chown, chmod, scp). The fix is one line: a literal `--` (the standard end-of-options marker) before the wildcard, or use `find -print0 | xargs -0 tar` instead.

## Vulnerabilities In One Place

- **Anonymous SMB with sensitive content** (CWE-284 + CWE-200): disable anonymous, never store credential material on any share.
- **Custom password wordlist as a file**: never archive plaintext password history. If audit requires it, store one-way hashes.
- **Cleartext credential delivery by email** (CWE-319): send a one-time reset link, not a password.
- **Security through obscurity** (CWE-1392): put pre-prod paths behind real authentication, basic auth at minimum.
- **CVE-2018-16763 in Cuppa CMS**: unmaintained CMS needs a maintenance owner or it needs to be removed. Set *allow_url_include = Off* globally.
- **tar wildcard injection in root cron** (CWE-78 + CWE-77): use `--` or `find -print0 | xargs -0` for any glob the script does not control.

## Key Takeaways

- **Chain thinking beats CVE thinking.** Five small findings, none individually critical, made the room. Real engagements look like this. Hygiene findings are the load-bearing ones.
- **enum4linux is the right first move against any open Samba port.** SID-to-username resolution converts every downstream auth step from a two-axis brute force into a one-axis one.
- **A wordlist on disk is sometimes the loot itself.** Read every file on every accessible share, even the ones that look operational.
- **Mailbox compromise is account compromise.** Every account whose reset flow lands in the mailbox you just compromised is now yours, and there is no fix that does not start with *"do not send credentials by email."*
- **searchsploit is the fastest CMS-version-to-CVE pipeline available.** Any version banner on a public-facing page deserves a searchsploit lookup before anything else.
- **Wildcards on the command line of a tool that interprets dashes are a privesc primitive.** Audit cron and systemd timers for any glob touching a service-writable path.

The full phase-by-phase walkthrough with screenshots lives in the [portfolio writeup](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-10-tryhackme-skynet).
