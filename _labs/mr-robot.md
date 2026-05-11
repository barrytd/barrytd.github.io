---
title: "Mr Robot"
date: 2026-05-01
platform: TryHackMe
difficulty: Medium
category: offensive
summary: "robots.txt to a leaked wordlist, WordPress username enumeration plus Hydra brute force, theme-editor PHP reverse shell, unsalted MD5 cracked on CrackStation, and SUID nmap interactive mode to root."
---

## Overview

Mr Robot is a medium-difficulty TryHackMe room themed after the TV show. Three keys, three lessons: how a robots.txt file becomes a sitemap for attackers, why WordPress admin equals web-server shell, and what happens when someone leaves the SUID bit on a binary old enough to still have an interactive mode.

The chain is a really clean introduction to **WordPress methodology**, **online vs offline password attacks**, and **classic Linux privilege escalation**, three skill sets that come up in almost every entry-level engagement.

## Tools Used

- **nmap** for the port scan.
- **gobuster** for WordPress directory discovery.
- **hydra** for the WordPress login brute force.
- **pentestmonkey php-reverse-shell** as the payload after admin access.
- **netcat** to catch the callback.
- **CrackStation** (online lookup service) for the MD5 hash.
- **find** for SUID enumeration.

## Methodology

**Step 1 - Service enumeration.** A standard port scan returns SSH on 22, HTTP on 80, and HTTPS on 443. The homepage is a stylized Mr Robot intro page with no obvious entry point.

```bash
nmap -p- -sV <TARGET_IP>
```

**Step 2 - robots.txt is not access control.** /robots.txt is a plaintext file that web servers serve at the root of any site to tell *web crawlers* (Google's bot, Bing's bot) which paths to skip. The key insight: it is a hint to *friendly* crawlers, not an access control. Anything you list there is publicly readable, and any attacker doing basic recon will fetch it first.

The Mr Robot box's /robots.txt lists two entries: a flag file (free for the asking) and a custom wordlist named *fsocity.dic* seeded with terms from the show.

**Step 3 - WordPress discovery.** A gobuster directory scan surfaces /wp-login.php, /wp-admin, /wp-content, and the rest of the standard WordPress install. Every WordPress box has the same logical entry point: the admin login.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Step 4 - Username enumeration via error-message differential.** Before brute-forcing a password, you need a valid username. WordPress hands one over for free through an old, well-documented behavior: the login form returns a *different error string* for "wrong password" than for "wrong username." Submitting a junk username returns *"Invalid username."* Submitting a real username with no password returns *"The password you entered for the username elliot is incorrect."* The text *"for the username elliot"* is the leak.

**Step 5 - Hydra against /wp-login.php.** The downloaded *fsocity.dic* contains a lot of duplicates. Deduplicating it shrinks the file dramatically and makes Hydra orders of magnitude faster.

```bash
sort -u fsocity.dic > fsocity_unique.dic

hydra -l elliot -P fsocity_unique.dic <TARGET_IP> http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR"
```

The failure marker *ERROR* is what tells Hydra a given password attempt failed. When the response does not contain *ERROR*, Hydra has a hit.

**Step 6 - WordPress admin equals shell.** Logging in lands on the admin dashboard. The Appearance menu exposes the **theme Editor**, which lets an administrator rewrite any PHP file in any installed theme directly from the browser. That is the code execution primitive needed for a shell. Overwriting the **404 Template** (404.php) with the *pentestmonkey* PHP reverse shell, then visiting any nonexistent URL inside the theme path, triggers the reverse shell to call back.

```bash
nc -lvnp 4444
# in browser, visit any nonexistent path under the theme to trigger 404.php
```

A reverse shell lands as the web user.

**Step 7 - Find and crack the MD5.** Inside /home/robot, two files are present: *key-2-of-3.txt* (locked, root-owned) and *password.raw-md5* (readable). The hash is a *raw MD5*: unsalted, fast-to-compute. Three problems: MD5 is broken for passwords, raw means unsalted, and a public lookup service like **CrackStation** maintains precomputed tables of common passwords to their MD5 hashes.

```bash
cat /home/robot/password.raw-md5
# paste the hash into crackstation.net
```

The plaintext returns immediately. There is no actual cracking; the lookup is in a database.

**Step 8 - Pivot to robot.**

```bash
su robot
# password from CrackStation
```

**Step 9 - SUID nmap from another era.** The next move is to look for **SUID** binaries. The *SUID bit* (Set User ID) on a Unix file means the program runs as its *owner's* user, not as the user who launched it.

```bash
find / -perm -4000 -type f 2>/dev/null
```

The list includes */usr/local/bin/nmap*. Modern nmap (version 7.x) has no shell escape. The version installed here is **3.81**, which still supports an *--interactive* mode removed years ago.

```bash
nmap --interactive
nmap> !sh
```

A root shell, in two commands.

## Key Takeaways

- **robots.txt is not access control.** Anything listed there is fetched by attackers first, before they think about exploitation.
- **Username enumeration is a 100x multiplier on a brute force.** A login page that distinguishes "wrong user" from "wrong password" hands the attacker the entire username axis for free.
- **Admin in WordPress equals shell on the host.** Treat WP admin credentials as web-server-user shell access in blast radius.
- **Unsalted MD5 belongs to the previous decade.** A lookup service like CrackStation resolves every common plaintext instantly.
- **SUID on a complex binary is almost always a privesc bug waiting to happen.** Run `find / -perm -4000` on every new Linux foothold.

## What a Defender Should Do

- Never list sensitive paths in robots.txt. Use authentication.
- Return a single generic error string for every failed login: *"Invalid login credentials."* No hint about which field was wrong.
- Set `DISALLOW_FILE_EDIT` and `DISALLOW_FILE_MODS` in *wp-config.php* on every production WordPress install. That single setting is the most effective hardening control.
- Store passwords with **Argon2id** or **bcrypt** with a per-user random salt and a deliberately slow work factor. Never raw MD5.
- Remove the SUID bit from anything that does not absolutely require it: `chmod u-s /usr/local/bin/nmap`. For the rare cases where real raw-socket access is needed, use Linux capabilities instead.
