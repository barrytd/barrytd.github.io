---
layout: post
title: "Ignite: When the Install Page Tells the Attacker Everything"
date: 2026-05-01
categories: [offensive-security, ctf]
tags: [cms, rce, cve-2018-16763, php, credential-reuse, methodology]
excerpt: An easy TryHackMe room that is really a credential-reuse story wearing an RCE costume. The CMS welcome page volunteers its version, admin path, and default credentials. A patched-but-not-deployed CMS hands over an unauthenticated shell. The actual privesc happens because the database password and the system root password are the same string.
---

> Companion to my [security-lab-portfolio writeup](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-01-tryhackme-ignite). Sanitized: no flag values or credentials.

## The Setup

A standard port scan returns exactly one open port: HTTP on 80 (Apache 2.4.18 on Ubuntu).

![Nmap]({{ "/assets/images/ignite/01-nmap-scan.png" | relative_url }})

65534 closed ports rule out the usual SSH/SMB/database-on-the-side surprises and force the engagement straight at the web app.

## The Landing Page Hands Over The Recon

Browsing to the target loads a getting-started page for **Fuel CMS** that *openly advertises*:

![Fuel CMS landing]({{ "/assets/images/ignite/02-webpage.png" | relative_url }})

- The CMS name and version: *Welcome to Fuel CMS, Version 1.4*
- The admin panel location: */fuel/*
- The default credentials: documented in the install guide right on the page
- The location of the database config file: */fuel/application/config/database.php*

This is the single biggest **information disclosure** I have seen in any beginner room, and it is a really pure example of why install pages should never survive into production. Modern attackers do not "fingerprint" applications; they grep CVE databases by version string. A version printed on a public-facing page is an invitation for an automated scan to come back with a working exploit.

## robots.txt Confirms It

/robots.txt contains a single line: *Disallow: /fuel/*. That corroborates the admin path the landing page already disclosed.

![robots.txt]({{ "/assets/images/ignite/03-robots-txt.png" | relative_url }})

Same lesson as the Mr Robot post: robots.txt is *a hint to crawlers, not an access control.*

## CVE-2018-16763: The create_function Sink

Fuel CMS 1.4.1 is affected by **CVE-2018-16763**, a public **Remote Code Execution** vulnerability with a published exploit.

The bug is in */fuel/pages/select/*, which accepts a *filter* query parameter and concatenates it directly into PHP's *create_function()* call. **create_function** is an old, deprecated PHP feature that compiles a string into a runnable function at runtime. Concatenating user input into it is the textbook **CWE-94: Code Injection** sink. An attacker closes the intended expression with a parenthesis and appends arbitrary PHP, which the server then runs.

A simple Python 2 wrapper script (Exploit-DB 47138) automates the URL encoding and command running:

![Exploit script]({{ "/assets/images/ignite/04-exploit-code.png" | relative_url }})

Running it with *ls* at the prompt confirms code execution as the web user:

![RCE confirmed]({{ "/assets/images/ignite/05-rce-confirmed.png" | relative_url }})

The fix: upgrade Fuel CMS to 1.4.4 or later, which stops using *create_function*. Better: stop deploying unmaintained CMS software. *create_function* itself was deprecated in PHP 7.2 and *removed in PHP 8*, so any project still depending on it has bigger problems.

## A Reverse Shell, Then The Config File

Sending a bash reverse shell command through the exploit prompt and catching the callback with netcat lands an interactive session as **www-data**.

The Fuel install guide on the landing page told me exactly where the database credentials live. Reading *fuel/application/config/database.php*:

![database.php]({{ "/assets/images/ignite/06-database-config.png" | relative_url }})

The MySQL root password is in plaintext inside the web root. Two problems:

1. **Plaintext credentials in a web-served directory.** Any file-read vulnerability (LFI, exposed *.git*, a misconfigured backup) exposes the database credentials.
2. **The credentials *are* in a place the web user could read by design**, so the moment the web user is compromised, so is the database.

The fix is to move secrets *out of the web root*. Read database credentials from environment variables loaded by systemd, from a real secrets manager (Vault, AWS Secrets Manager), or from a 0640-mode file outside any directory the web server serves.

## The Privesc Was The Password

The string in *database.php* is the MySQL administrative password. By itself, that is a database credential, not a system credential. But on small installs it is extremely common for the same operator password to be reused for the system *root* account.

*su root* with the recovered MySQL password succeeds on the first try.

That is the entire privesc. No kernel exploit, no SUID bug, no misconfigured cron. Just **password reuse across trust boundaries**, which is the single most common privesc vector in real engagements.

The fix is generated-and-stored-in-a-password-manager, *independent* credentials for every privileged account. Reusing a password across MySQL admin and system root collapses two trust boundaries into one.

## Vulnerabilities In One Place

- **CVE-2018-16763 - Fuel CMS unauth RCE** (CWE-94): upgrade or remove. *create_function* was deprecated for a reason.
- **Verbose version disclosure on the landing page** (CWE-200): delete install/getting-started pages once setup is complete. Strip *Server* and *X-Powered-By* headers.
- **Default credentials documented and active** (CWE-798 + CWE-521): force a password change on first login. Refuse to start the app until rotated.
- **Plaintext credentials in the web root** (CWE-256 + CWE-538): move secrets outside any web-served directory.
- **Password reuse between database admin and system root** (CWE-521): generate independent high-entropy passwords for every privileged account. Disable root password login in sshd.

## Key Takeaways

- **The whole chain in this room is a credential-reuse story dressed up as an RCE.** Even if the CMS were patched, the box would still fall the moment any other vulnerability surfaced *database.php*. Operational hygiene is the load-bearing control, not the patch level.
- **A "version 1.4" string on a landing page is a working exploit.** Modern attackers do not fingerprint, they grep CVE feeds. Any product version printed on a public-facing page is an invitation.
- **Default credentials in install documentation are themselves a vulnerability.** A welcome page that tells the attacker what credentials to try is functionally identical to leaving the panel unauthenticated.
- ***create_function* is a code-injection sink waiting to happen.** Any function that compiles strings into runnable code (*eval*, *create_function*, JavaScript *new Function*, Python *exec*) is one user-input touch away from RCE.
- **Always read every config file your foothold can reach.** The path from *www-data* to *root* in this room was a single *cat* command on a file the web user could read by design. After every web shell, before downloading enum scripts, enumerate readable config files for plaintext secrets.

The full per-phase walkthrough with screenshots lives on the [portfolio repo](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-01-tryhackme-ignite).
