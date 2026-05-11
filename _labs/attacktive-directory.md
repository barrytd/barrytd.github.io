---
title: "Attacktive Directory"
date: 2026-02-26
platform: TryHackMe
difficulty: Medium
category: offensive
summary: "Full Active Directory methodology: Kerberoasting, AS-REP Roasting, and BloodHound attack-path mapping ending at Domain Admin."
---

## Overview

Attacktive Directory is a medium-difficulty TryHackMe room that walks through the **standard Active Directory pentest methodology** end to end. The three pillars: **Kerberoasting** (request service tickets and crack them offline), **AS-REP Roasting** (request pre-auth-disabled accounts' encrypted timestamps and crack them offline), and **BloodHound** (a graph database of AD relationships that finds the shortest path to Domain Admin).

If you only ever do one AD room on TryHackMe, this is the one.

## Tools Used

- **nmap** for the port scan.
- **enum4linux** and **kerbrute** for user enumeration.
- **Impacket** suite: `GetNPUsers.py`, `GetUserSPNs.py`, `secretsdump.py`, `psexec.py`.
- **hashcat** for offline cracking.
- **BloodHound** for attack-path mapping.
- **SharpHound** to collect the data BloodHound visualizes.

## Methodology

**Step 1 - Service enumeration.** A standard Windows-domain port set: 53 (DNS), 88 (Kerberos), 135 (RPC), 139/445 (SMB), 389/636 (LDAP / LDAPS), 3268 (Global Catalog), 3389 (RDP), 5985 (WinRM).

```bash
nmap -p- -sV <TARGET_IP>
```

The presence of port 88 confirms it's a domain controller.

**Step 2 - User enumeration with kerbrute.** Active Directory leaks valid usernames through Kerberos. When you request a TGT for an unknown username, the response is *"client unknown"*. For a known username, the response is *"pre-auth required"*. **kerbrute** walks a wordlist and reports which usernames are valid.

```bash
./kerbrute userenum -d <DOMAIN> --dc <DC_IP> /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
```

This is the **username enumeration** step. Like SquirrelMail and WordPress, knowing valid usernames is the multiplier on every downstream attack.

**Step 3 - AS-REP Roasting.** Some user accounts in AD have *Do not require Kerberos preauthentication* set. For those accounts, anyone can request an AS-REP message (the initial Kerberos response) without proving they know the password. The AS-REP includes a portion encrypted with the user's password hash, which can be cracked offline.

```bash
impacket-GetNPUsers <DOMAIN>/ -dc-ip <DC_IP> -no-pass -usersfile valid_users.txt
```

The output is a hash string (hashcat mode 18200) for any account with pre-auth disabled.

**Step 4 - Crack the AS-REP hash.**

```bash
hashcat -m 18200 -a 0 asrep_hash.txt /usr/share/wordlists/rockyou.txt
```

The recovered plaintext is the user's password.

**Step 5 - Kerberoasting.** Some user accounts run services and are registered with **Service Principal Names (SPNs)**. Any domain user can request a service ticket (TGS) for a service registered to an SPN. The ticket is encrypted with the *service account's* NTLM hash, which means an authenticated low-privileged user can extract crackable hashes for every service account in the domain.

```bash
impacket-GetUserSPNs <DOMAIN>/<user>:<password> -dc-ip <DC_IP> -request
```

The output is a hash string (hashcat mode 13100) for every kerberoastable account.

**Step 6 - Crack the TGS hash.**

```bash
hashcat -m 13100 -a 0 tgs_hash.txt /usr/share/wordlists/rockyou.txt
```

Service accounts often have predictable passwords because they're set by humans years ago and never rotated. Recovery rates against Kerberoasted hashes are notoriously high.

**Step 7 - BloodHound for attack paths.** BloodHound is a graph database that ingests AD relationship data (group membership, ACL entries, session info, computer-account property changes) and presents shortest paths to high-value targets like Domain Admin.

```bash
# collect from a domain-joined Windows host:
./SharpHound.exe -c All

# load the resulting .zip into BloodHound, then query:
# "Shortest Paths to Domain Admins"
```

The graph reveals which accounts and groups are one or two hops away from Domain Admin, and which ACL misconfigurations enable each hop.

**Step 8 - Pivot through the recovered credentials.** Once a sufficiently-privileged account is compromised, **secretsdump** extracts every credential from the DC (NTDS.dit if local, or via DCSync if the account has *Replicating Directory Changes* rights):

```bash
impacket-secretsdump <DOMAIN>/<admin_user>:<password>@<DC_IP>
```

The output is the NTLM hashes of every account in the domain, including the krbtgt account, which enables Golden Tickets.

**Step 9 - Lateral movement.** With credentials, **psexec.py** (Impacket's open-source psexec) drops a SYSTEM shell on any domain-joined host that allows admin SMB access.

```bash
impacket-psexec <DOMAIN>/<admin>:<password>@<TARGET_IP>
```

## Key Takeaways

- **AD username enumeration is free over Kerberos.** Every engagement starts with kerbrute against the DC.
- **AS-REP Roasting and Kerberoasting are textbook AD attacks.** Both work without admin privileges and recover crackable hashes for downstream cracking.
- **Service accounts are the AD weak link.** They are set by humans, given strong privileges, and rarely rotated.
- **BloodHound finds the path you would never find by reading the documentation.** Real-world AD environments accumulate broken ACLs over years.
- **DCSync is a one-shot game over.** Any account with *Replicating Directory Changes* effectively owns the domain.

## What a Defender Should Do

- Disable *Do not require Kerberos preauthentication* on every account that doesn't strictly need it. AS-REP Roasting is a free win for attackers otherwise.
- Use managed service accounts (gMSAs) with auto-rotated 120-character passwords for every service that needs an SPN. Kerberoasting against a gMSA is computationally infeasible.
- Audit AD ACLs regularly with BloodHound from a defender's perspective. The graph that shows you the attack path also shows you which edges to break.
- Restrict *Replicating Directory Changes* to a tiny named group. The default *Domain Admins* + *Enterprise Admins* coverage is usually appropriate, but any other principal with this right is a critical finding.
- Enable AdminSDHolder protection and tier the admin model (Tier 0 / Tier 1 / Tier 2). Don't let DA accounts log on to workstations.
- Monitor for unusual TGS request patterns. A single user requesting service tickets for every SPN in the domain in two minutes is a Kerberoasting signature.
