---
title: "Active Directory - Full Domain Compromise"
date: 2026-02-24
platform: Self-built
difficulty: Hard
category: offensive
summary: "Built and attacked a home-lab Active Directory environment: NTLM dump, Pass-the-Hash, PSExec to SYSTEM, then krbtgt extraction and Golden Ticket forgery for persistent Domain Admin."
---

## Overview

This is a self-built **Active Directory pentest lab**, not a TryHackMe room. I stood up a Windows Server domain controller, joined a Windows 10 client, configured a service account and a domain admin, and then attacked the environment using the same techniques a real adversary would.

The chain demonstrates the full AD compromise lifecycle: **initial foothold → credential extraction → lateral movement → domain admin → Golden Ticket persistence**.

## Tools Used

- **Windows Server (DC)** and a **Windows 10 client** as the lab.
- **Impacket** for the attack tooling (psexec.py, secretsdump.py).
- **mimikatz** for in-memory credential extraction.
- **Pass-the-Hash** technique (no password required, just the hash).
- **wmic** and PowerShell for enumeration.
- **CrackMapExec / NetExec** for spraying credentials across the domain.

## Methodology

**Step 1 - Build the lab.** A minimal AD lab needs:

- One Windows Server (2019 or 2022) promoted to domain controller via *dcpromo / Install-ADDSForest*.
- A domain name (I used `lab.local`).
- One or more Windows 10 / 11 clients joined to the domain.
- A few user accounts of varying privilege: regular user, domain admin, service account with an SPN.

The lab matters because pen-testing your own AD is the only practical way to learn the attack lifecycle without authorization issues.

**Step 2 - Get an initial foothold.** Any of the standard initial-access vectors works: phishing-style social engineering (in a lab, just hand the user the malware), an unpatched service, exposed credentials in a network share. For the writeup I assumed a non-privileged shell on the Windows 10 client.

**Step 3 - Local credential extraction with mimikatz.** Mimikatz reads LSASS memory to extract every credential cached on the local box. SYSTEM access is required (which an admin shell on the client provides).

```cmd
privilege::debug
sekurlsa::logonpasswords
```

The output includes NTLM hashes for every account that has interactively logged on to this machine, including any domain admin who has ever RDPed in (a common lab mistake and a real-world disaster).

**Step 4 - Pass-the-Hash.** With an NTLM hash, the attacker can authenticate to other domain hosts without ever knowing the plaintext. Impacket's psexec accepts a hash directly.

```bash
impacket-psexec -hashes :<NTLM_HASH> <DOMAIN>/<user>@<TARGET>
```

The lack of a plaintext on the left of the colon and the `:<HASH>` format is the *"pass the hash"* idiom. SMB authentication does not require knowing the password, only the hash.

**Step 5 - Lateral movement.** Once one host is compromised, mimikatz again extracts whatever credentials are cached on that host. Repeat until a Domain Admin or a service account with replication privileges appears.

```bash
impacket-psexec -hashes :<HASH> <DOMAIN>/<da_user>@<DC_IP>
```

A SYSTEM shell on the domain controller follows.

**Step 6 - NTDS.dit dump with secretsdump.** The Domain Controller stores every account's hash in `C:\Windows\NTDS\NTDS.dit`. With admin access to the DC, **secretsdump** can extract them.

```bash
impacket-secretsdump <DOMAIN>/<da>:<password>@<DC_IP>
```

For a more stealthy approach, **DCSync** uses MS-DRSR (the legitimate replication protocol) to ask the DC to send its credentials, which is what domain controllers do to each other normally. Any account with *Replicating Directory Changes* and *Replicating Directory Changes All* can DCSync.

**Step 7 - Extract krbtgt hash.** The crown jewel is the **krbtgt account hash**. krbtgt is the account whose key signs every Kerberos ticket-granting ticket (TGT) in the domain. Knowing its hash means being able to forge TGTs for any user.

**Step 8 - Forge a Golden Ticket.** A **Golden Ticket** is a TGT signed by the krbtgt hash. The attacker constructs it offline with mimikatz and presents it to the DC. The DC validates the signature, sees krbtgt's signature, and trusts the ticket.

```cmd
mimikatz # kerberos::golden /domain:lab.local /sid:S-1-5-21-... /rc4:<krbtgt_hash> /user:Administrator /ptt
```

`/user:Administrator` forges a ticket *as* Administrator, even if the account doesn't exist. `/ptt` injects the ticket into the current Kerberos cache.

**Step 9 - Persistent Domain Admin.** From the moment the Golden Ticket is injected, every operation that uses Kerberos authentication (PSExec, WMI, remote PowerShell, SMB) treats the attacker as Administrator. The ticket is valid by default for 10 years, and rotating the krbtgt password twice is the only thing that invalidates it.

## Key Takeaways

- **Building your own AD lab is the single most valuable thing a beginner can do** to learn enterprise security. Books cannot replicate the experience.
- **mimikatz + Pass-the-Hash + PSExec is the classic AD lateral movement triplet.** Defenders should assume this combination is in play in every breach.
- **DCSync is the modern stealthy alternative to NTDS.dit dumping.** It does not require touching the DC's filesystem or services.
- **Golden Tickets are persistent and very hard to detect.** Once issued, only krbtgt password rotation (twice, with a wait between) revokes them.
- **The krbtgt hash is the single most important secret in any AD environment.** Protect it like the root CA private key.

## What a Defender Should Do

- Tier the admin model (Tier 0 / Tier 1 / Tier 2). DA accounts should never log on to anything below Tier 0 (DCs, ADCS, ADFS). Workstation/server compromise should never cascade up.
- Enable Credential Guard on Windows 10/11 to protect LSASS from mimikatz-style memory reads.
- Restrict membership in privileged groups (Domain Admins, Enterprise Admins, Backup Operators, Print Operators). The fewer accounts, the smaller the blast radius.
- Rotate the krbtgt password regularly (Microsoft's recommendation is yearly, more often after any compromise). Invalidate Golden Tickets with the standard procedure: rotate, wait 10 hours, rotate again.
- Run **PingCastle** or **PurpleKnight** (free AD audit tools) quarterly. They find the ACL misconfigurations BloodHound also surfaces.
- Monitor for `secretsdump` indicators: VSS shadow copy creation, suspicious LDAP replication requests, unusual service installations on the DC. Sysmon EventIDs 7, 8, and 11 plus security 4624/4672/4768/4769 are the core dataset.
