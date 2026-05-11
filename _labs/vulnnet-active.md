---
title: "VulnNet: Active"
date: 2026-02-27
platform: TryHackMe
difficulty: Medium
category: offensive
summary: "Unauthenticated Redis to a Responder NTLM capture, cracked the hash offline, used the credentials for SMB share abuse, then GodPotato to SYSTEM."
---

## Overview

VulnNet: Active is a medium-difficulty Windows TryHackMe room that chains four classic enterprise-network patterns: **unauthenticated Redis** as the initial info leak, **Responder NTLM hash capture** by forcing the target to authenticate to the attacker, **offline hash cracking** with hashcat, and **GodPotato** to escalate from a service account to SYSTEM via *SeImpersonatePrivilege*.

## Tools Used

- **nmap** for the port scan.
- **redis-cli** for the Redis enumeration.
- **Responder** to capture NTLMv2 hashes by impersonating SMB/HTTP services.
- **hashcat** to crack the captured hash offline.
- **smbclient** and **smbmap** for share enumeration.
- **GodPotato** for the SeImpersonate → SYSTEM upgrade.

## Methodology

**Step 1 - Service enumeration.** A standard nmap scan returns the usual Windows ports plus a Redis instance on port 6379, which is unusual for a Windows box.

```bash
nmap -p- -sV <TARGET_IP>
```

**Step 2 - Unauthenticated Redis.** Redis by default has no authentication. Anyone who can connect to the port can read and write the data store.

```bash
redis-cli -h <TARGET_IP>
```

Inside the Redis shell, the attacker can enumerate keys, read configuration, and (depending on Redis version and config) write files to disk through the RDB persistence mechanism.

**Step 3 - Force the target to authenticate to me.** The room is set up so that a writable share or an SMB-aware file on the target can be made to point at the attacker. **Responder** is the tool for this: it impersonates SMB, HTTP, and other Windows authentication services on the local segment, and captures any NTLM authentication that lands on it.

```bash
sudo responder -I <interface> -A
```

The `-I` is the network interface to listen on. `-A` is *analyze mode* (passive listening). Triggering an SMB authentication from the target (a file:// URL, an UNC path, a misconfigured share) causes the target to send its NTLMv2 hash to Responder, which logs it to disk.

**Step 4 - Crack the hash offline.** NTLMv2 hashes are hashcat mode 5600.

```bash
hashcat -m 5600 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

`-m 5600` is NTLMv2. `-a 0` is straight dictionary attack. The recovered plaintext is the service account password.

**Step 5 - SMB share abuse.** With valid credentials, enumerate accessible shares.

```bash
smbmap -H <TARGET_IP> -u <user> -p <password>
```

The output lists shares the credentials can read or write. One share contains a binary that runs as a more-privileged user, plus space to drop attacker files.

**Step 6 - GodPotato to SYSTEM.** The compromised account has **SeImpersonatePrivilege** (which is the default for IIS, MSSQL, and many service accounts in Windows).

```cmd
whoami /priv
```

GodPotato is the modern successor to RottenPotato and JuicyPotato. It exploits the same SeImpersonate technique: trick a SYSTEM service into authenticating to a local RPC endpoint controlled by the attacker, capture the resulting impersonation token, and use it to spawn a SYSTEM process.

```cmd
copy \\<TARGET>\share\GodPotato-NET4.exe C:\Users\<user>\Downloads\
C:\Users\<user>\Downloads\GodPotato-NET4.exe -cmd "cmd /c whoami"
```

The output reads **NT AUTHORITY\SYSTEM**, confirming the upgrade. From SYSTEM, both flags are readable.

## Key Takeaways

- **Unauthenticated Redis is a critical exposure.** Redis on the default port with no AUTH directive is the single most common cause of leaked enterprise data. *redis-cli -h* against any new internal IP is a worthwhile recon step.
- **Responder is the most productive tool on any internal pentest.** Hostname typos, mistyped UNC paths, and broken DNS all cause Windows to fall back to LLMNR / NBT-NS / MDNS, which Responder will happily answer.
- **NTLMv2 hashes are hashcat mode 5600.** The hashes are crackable in seconds if the password is in rockyou, in days if it's not.
- **SeImpersonatePrivilege equals SYSTEM** on any version of Windows from XP onward. GodPotato is the current go-to.

## What a Defender Should Do

- Authenticate every Redis instance (`requirepass` in redis.conf) and bind it to *localhost only* unless there's a hard requirement for remote access. Cache eviction issues at AWS, Imgur, and many others all started with unauthenticated Redis.
- Disable LLMNR and NBT-NS via Group Policy on the entire enterprise. They are 1990s name-resolution protocols that exist almost exclusively to be poisoned by attackers.
- Enforce SMB signing on all server and client endpoints. SMB signing prevents relay attacks against captured authentication.
- Remove SeImpersonatePrivilege from service accounts that don't strictly need it. The IIS application pool identity is the most common offender.
- Use modern Kerberos-only authentication where possible. NTLMv2 hashes are crackable; Kerberos tickets are time-bounded.
- Run AppLocker or Windows Defender Application Control to block unknown binaries from running in service accounts' contexts. GodPotato is a known signature.
