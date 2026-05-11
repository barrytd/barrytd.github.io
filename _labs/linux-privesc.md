---
title: "Linux PrivEsc"
date: 2026-03-01
platform: TryHackMe
difficulty: Medium
category: offensive
summary: "18 Linux privilege escalation techniques: MySQL UDF, sudo and SUID abuse, cron hijacking, NFS no_root_squash, wildcard injection, and Dirty COW (CVE-2016-5195)."
---

## Overview

Linux PrivEsc is a medium-difficulty TryHackMe room that is really a 18-technique catalog. Every privesc category you'll encounter in real engagements is represented: misconfigured sudo, SUID abuse, environment variable manipulation, cron job hijacking, NFS no_root_squash exports, wildcard injection in shell scripts, kernel exploits, MySQL UDF, and writable /etc/passwd.

This is the single best Linux privesc reference room TryHackMe offers, and the techniques covered map directly to what you'll see on real boxes.

## Tools Used

- **find** for SUID/SGID enumeration.
- **sudo -l** for sudo policy enumeration.
- **GTFOBins** as the reference for binary shell escapes.
- **LinPEAS** (or `linux-exploit-suggester`) for kernel-version exploit discovery.
- **gcc** for compiling the Dirty COW exploit.

## Methodology

The room presents 18 separate privesc paths. I'll group them by category since that's how you'll think about them in a real engagement.

### Sudo abuse (4 techniques)

**Step 1 - sudo -l first.** Every Linux foothold starts with `sudo -l`. The output reveals which commands the current user can run as root, and with what flags.

```bash
sudo -l
```

The four sudo-abuse patterns in this room:

- **sudo with the binary's documented shell escape.** *less, find, awk, vim, etc.* all have shell escapes. Cross-reference against **GTFOBins**.
- **sudo LD_PRELOAD.** If sudoers preserves the LD_PRELOAD environment variable (`Defaults env_keep += "LD_PRELOAD"`), the attacker can write a malicious .so file with a static constructor and use sudo to run any allowed command, which loads the malicious library as root.
- **sudo CVE-2019-14287.** A logic bug in sudo where `sudo -u#-1` runs as UID 0 even when the sudoers policy says *anyone except root*.
- **sudo with a relative path or PATH manipulation.** Wrapper scripts that invoke binaries by name (`cp` instead of `/bin/cp`) inherit the user's PATH and can be hijacked.

### SUID abuse (3 techniques)

**Step 2 - Find SUID files.**

```bash
find / -perm -4000 -type f 2>/dev/null
```

The three SUID patterns in this room:

- **SUID on a binary with a documented shell escape.** GTFOBins again.
- **SUID on a custom script or compiled binary** that calls another binary by name. Same PATH-hijack pattern as the sudo case.
- **SUID bash with -p.** Discussed extensively in Overpass 2 Hacked.

### Cron and scheduled task abuse (3 techniques)

**Step 3 - Read /etc/crontab.**

```bash
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/
```

Patterns:

- **Writable script in a root cron.** The script the cron runs is owned by a lower-privileged user. Edit it, wait one minute.
- **Wildcard injection in a cron'd command.** `tar`, `chown`, `chmod` with wildcards in a directory the attacker can write into.
- **PATH manipulation in a root cron.** If the cron's command uses an unqualified binary name and PATH includes a writable directory, plant a binary there.

### Capabilities (1 technique)

**Step 4 - List capabilities.**

```bash
getcap -r / 2>/dev/null
```

Linux capabilities split root's powers into smaller pieces. *cap_setuid+ep* on a binary like Python is functionally equivalent to setting it SUID-root. Less common than SUID but worth checking.

### Writable /etc/passwd (1 technique)

**Step 5 - Check passwd permissions.**

```bash
ls -la /etc/passwd
```

If */etc/passwd* is writable by a non-root user (which it shouldn't be, ever), the attacker can append a new root entry with a known password hash.

```bash
echo 'attacker::0:0:::/bin/bash' >> /etc/passwd
# password is empty in this example; for a real one, hash a chosen password with openssl
```

### NFS no_root_squash (1 technique)

**Step 6 - Check exports.**

```bash
cat /etc/exports
```

`no_root_squash` is the option that tells the NFS server *"trust the client's root user as if they were our root user."* If a share is exported with no_root_squash and the attacker controls a client, they can create a SUID-root binary on the share and execute it from the target.

### Kernel exploits (2 techniques)

**Step 7 - Check kernel version.**

```bash
uname -a
```

The room demonstrates **Dirty COW (CVE-2016-5195)**, a race condition in the Linux kernel's copy-on-write handling that lets a non-privileged user write to a read-only memory mapping. The classic exploitation is to overwrite a SUID binary on disk.

```bash
gcc -pthread dirtycow.c -o dirtycow -lcrypt
./dirtycow
```

Kernel exploits are a last resort because they're loud, often unstable, and can crash the box.

### MySQL UDF (1 technique)

**Step 8 - MySQL running as root.** If MySQL is running as the root user (which it should not be on modern installs), and the attacker has MySQL admin credentials, they can write a User Defined Function in C, compile it, drop it in MySQL's plugin directory, and call it from SQL to execute arbitrary code as root.

### Service misconfigurations (2 techniques)

**Step 9 - Examine running services.** Services running as root that read configuration from world-writable files give the attacker code execution as root by editing the config.

### File permissions on sensitive files (1 technique)

**Step 10 - Find writable system files.**

```bash
find / -writable -type f 2>/dev/null | grep -v /proc
```

## Key Takeaways

- **`sudo -l` and `find / -perm -4000` are the two first commands on every Linux foothold.** Together they cover the majority of privesc paths.
- **GTFOBins is the single most valuable bookmark.** Memorize the names: less, find, awk, vim, nano, perl, python, ruby, ftp, more, env, expect.
- **Cron is a privesc treasure trove** when any of its scripts, wildcards, or PATH dependencies touch a non-root-writable area.
- **NFS no_root_squash is a one-shot privesc** when the attacker controls the NFS client.
- **Kernel exploits are powerful but loud.** Dirty COW reliably wins, but it can crash the box, and a defender's kernel-version inventory catches the precondition.

## What a Defender Should Do

- Audit sudoers regularly. Remove any entry that points at a GTFOBins binary unless the operational need is documented.
- Remove SUID bits from binaries that don't strictly need them. Use Linux capabilities as a narrower alternative when a privilege is needed.
- Run MySQL, PostgreSQL, Redis, and other database services as dedicated low-privilege accounts. Never as root.
- Restrict NFS exports with *root_squash* (the default) and IP-based access controls.
- Patch the kernel. Dirty COW is from 2016. There is no excuse to be vulnerable to it in 2024+.
- Set `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/sudoers`, and the cron directories to 0644 / 0600 / 0440 as appropriate. Run file integrity monitoring (AIDE, Tripwire, Wazuh FIM) on those paths.
- Enable auditd rules for SUID file creation and writes to critical config files. Sysmon Linux is now production-grade and a strong choice.
