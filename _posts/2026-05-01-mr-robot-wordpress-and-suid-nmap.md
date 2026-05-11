---
layout: post
title: "Mr Robot: robots.txt to WordPress to a SUID Binary From a Previous Decade"
date: 2026-05-01
categories: [offensive-security, ctf]
tags: [wordpress, hydra, hash-cracking, suid, privesc, methodology]
excerpt: A medium-difficulty TryHackMe room based on the show. Three keys, three lessons: how a robots.txt file becomes a sitemap for attackers, why WordPress admin equals web-server shell, and what happens when someone leaves the SUID bit on a binary old enough to still have an interactive mode.
---

> Companion to my [security-lab-portfolio writeup](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-01-tryhackme-mr-robot). Sanitized: no flag values, passwords, or hash plaintexts.

## The Setup

A Nessus baseline scan returns 24 findings, mostly informational fingerprints. The map then narrows to the web app, which is what every Mr Robot themed box is really about.

![Nmap]({{ "/assets/images/mr-robot/02-nmap-scan.png" | relative_url }})

Three open ports: SSH on 22, HTTP on 80, HTTPS on 443. The homepage is a stylized Mr Robot intro page with no obvious entry point. The actual lead is somewhere else.

## Why robots.txt Matters

**robots.txt** is a plaintext file that web servers serve at the root of any site to tell *web crawlers* (Google's bot, Bing's bot) which paths to skip. The key insight: it is a *hint to friendly crawlers*, not an access control. Anything you list there is **publicly readable**, and any attacker doing basic recon will fetch it first.

The Mr Robot box's /robots.txt lists two entries:

- *key-1-of-3.txt* (a flag file, free for the asking)
- *fsocity.dic* (a custom wordlist seeded with terms from the show)

![robots.txt]({{ "/assets/images/mr-robot/04-robots-txt.png" | relative_url }})

The first key is exposed directly. The wordlist becomes the ammunition for the next step.

## Gobuster Confirms It Is WordPress

A directory brute force surfaces */wp-login.php, /wp-admin, /wp-content,* and the rest of the standard WordPress install. Every WordPress box has the same logical entry point: the admin login.

![Gobuster]({{ "/assets/images/mr-robot/06-gobuster-enumerate.png" | relative_url }})

## Username Enumeration via Error-Message Differential

Before brute-forcing a password, you need a valid username. WordPress hands one over for free through an old, well-documented behavior: the login form returns a *different error string* for "wrong password" than for "wrong username."

![Username enumeration]({{ "/assets/images/mr-robot/09-username-enumeration.png" | relative_url }})

Submitting a junk username returns *"Invalid username."* Submitting a real username with no password returns *"The password you entered for the username elliot is incorrect."* The text *"for the username elliot"* is the leak. This is **CWE-204: Response Discrepancy**.

The fix is to return a single generic error for every failed login: *"Invalid login credentials,"* with no hint about whether the username, the password, or both were wrong.

## Hydra Trims the Wordlist and Crushes It

The downloaded *fsocity.dic* contains a lot of duplicate entries. Deduplicating it (`sort -u fsocity.dic > fsocity_unique.dic`) shrinks it from roughly 858k lines to about 11k unique words and makes Hydra orders of magnitude faster.

Hydra in *http-post-form* mode is told the URL, the form parameters, and the failure-text marker (the literal word *ERROR* in the response). When the response does not contain *ERROR*, it has a hit.

A wordlist this small and a target this old loses quickly. The exact password is omitted from this post.

## WordPress Admin Equals Shell on the Host

Logging in lands on the admin dashboard. The header reads *WordPress 4.3.1 running Twenty Fifteen theme*. The Appearance menu in the sidebar exposes the **theme Editor**, which lets an administrator rewrite any PHP file in any installed theme directly from the browser.

![Dashboard]({{ "/assets/images/mr-robot/11-wp-admin-dashboard.png" | relative_url }})

That is the code execution primitive needed for a shell. Overwriting the **404 Template** (404.php) with the *pentestmonkey* PHP reverse shell, then visiting any nonexistent URL inside the theme path, triggers the reverse shell to call back.

![Theme editor]({{ "/assets/images/mr-robot/12-reverse-shell-script-in-theme-editor.png" | relative_url }})

This is the textbook *"compromise an admin account = web-server-user shell"* path on WordPress. The fix is one line in *wp-config.php*: `define('DISALLOW_FILE_EDIT', true);`. That single setting is the most effective hardening control on any production WordPress install.

A netcat listener catches the callback as **daemon**, the same low-privileged web user.

![Reverse shell]({{ "/assets/images/mr-robot/13-reverse-shell.png" | relative_url }})

## A Bad Way to Store a Password

Inside /home/robot, two files are readable: *key-2-of-3.txt* (locked, root-owned) and *password.raw-md5* (world-readable, despite the name suggesting otherwise).

Catting the latter reveals a `user:hash` line in **raw MD5** format. Three problems here:

1. **MD5 is broken for passwords.** A modern GPU tries billions of MD5 guesses per second. The algorithm was designed for *file integrity*, not for credential storage.
2. **Raw means unsalted.** A *salt* is a per-user random value mixed into the hash so that two users with the same password produce different hashes. Without it, every common password collapses to the same value and rainbow tables work.
3. **The file is readable by another user.** Even if the hash were strong, leaking the hash file to a low-privileged account is half a compromise.

CrackStation (a free online lookup service that maintains a precomputed table of common-password-to-MD5 mappings) returns the plaintext immediately. There is no actual *cracking* happening; the lookup is in a database.

The fix is **Argon2id** or **bcrypt** with a per-user random salt and a deliberately slow work factor.

## SUID Nmap From Another Era

After *su robot*, the next move is to look for **SUID** binaries. The *SUID bit* (Set User ID) on a Unix file means the program runs as its *owner's* user, not as the user who launched it. SUID is the mechanism behind tools like *passwd* and *sudo*, which need root to do their job but are launched by ordinary users.

A `find / -perm -4000 -type f` surfaces an entry that does not belong: */usr/local/bin/nmap*. Modern nmap (version 7.x) has no shell escape. The version installed here is **3.81**, which still supports an *--interactive* mode that was removed years ago.

Old nmap's interactive mode accepts a *!* prefix to spawn shell commands. Because nmap is SUID-root, the spawned shell inherits root's privileges:

```
find / -perm -4000 -type f 2>/dev/null
nmap --interactive
nmap> !sh
```

A root shell, in two commands. The third key file then drops out.

This is **CWE-250 + CWE-732**: execution with unnecessary privileges, plus dangerous default file permissions. The fix is `chmod u-s /usr/local/bin/nmap`. nmap has not needed SUID in over a decade.

## Vulnerabilities In One Place

- **Sensitive file disclosure via robots.txt** (CWE-200): never list sensitive paths in robots.txt. Use authentication.
- **Username enumeration via response discrepancy** (CWE-204): return a single generic error string for every failed login.
- **No brute-force protection** (CWE-307): rate limit, install *Wordfence* or *Limit Login Attempts Reloaded*, enforce MFA.
- **WordPress in-dashboard file editor enabled** (CWE-732 + CWE-94): set *DISALLOW_FILE_EDIT* in wp-config.php on every production install.
- **Unsalted MD5 password file in a readable location** (CWE-916 + CWE-759): use Argon2id or bcrypt, restrict the file to 0600, never store credentials in a sibling user's home.
- **SUID on a binary with a documented shell escape** (CWE-250 + CWE-732): remove the SUID bit, use Linux capabilities for the rare cases real raw-socket access is needed.

## Key Takeaways

- **robots.txt is not access control.** Anything listed there is fetched by attackers first, before they think about exploitation.
- **Username enumeration is a 100x multiplier on a brute force.** A login page that distinguishes "wrong user" from "wrong password" hands the attacker the entire username axis for free.
- **Admin in WordPress equals shell on the host.** Treat WP admin credentials as web-server-user shell access in blast radius. Disable the file editor by default.
- **Unsalted MD5 belongs to the previous decade.** A lookup service like CrackStation resolves every common plaintext instantly. Argon2id or bcrypt is the floor for new code.
- **SUID on a complex binary is almost always a privesc bug waiting to happen.** Anything with a shell escape (less, nano, vi, awk, find, perl, python, *and nmap on an old enough version*) becomes one-line root the moment its SUID bit is set. Run `find / -perm -4000` on every new Linux foothold.

The full per-phase walkthrough with screenshots lives on the [portfolio repo](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-01-tryhackme-mr-robot).
