---
title: "My First SIEM Investigation - What I Learned Working Through BOTSv2 in Splunk"
date: 2026-05-12
platform: TryHackMe
difficulty: Hard
category: blue-team
summary: "Three days inside the BOTSv2 dataset on TryHackMe's Splunk 2 room. Four scenarios, one fake brewery, and a lot of Base64. Notes on what clicked, what didn't, and what I'd tell a beginner before they start."
---

## Why This Was The Hardest Lab I've Done

Three days. That's how long Splunk 2 on TryHackMe took me. It is the **longest single lab** I've finished and the one I learned the most from. The format is unusual: instead of one box with one entry point and one privesc, BOTSv2 is a giant dataset with **four independent investigations** stitched into a single room. You are Alice Bluebird, a SOC analyst at a fictional brewery called Frothly, and the dataset contains the full telemetry of everything that happened to the company in one bad week. Your job is to keep finding things.

The four storylines:

- **Insider threat.** An employee browsing the competitor's website and emailing them company documents.
- **Web attacks.** A vulnerability scanner, a SQL injection, an XSS payload, and a fake account created with a misspelled name.
- **Ransomware.** A Mac that got encrypted. A USB stick that delivered the malware to another Mac before that.
- **APT.** A North Korean group called Taedonggang dropping spear phishing on the network, using HTTPS with self-signed certs, registry-stored C2 URLs, and a scheduled task with a Base64-encoded PowerShell payload.

Each one taught me a different piece of the SOC analyst job. I want to write down the parts that clicked because *those* are the lessons that transfer to the next investigation, not the answers I found.

## What Splunk Actually Is

Before this lab I had a vague idea of what SIEM meant. I came out with a concrete one.

**Splunk** is a database that holds logs. Every line in every log file from every machine in the environment gets indexed into Splunk by **sourcetype** (the kind of log it is) and **index** (the bucket it lives in). Once it is in Splunk, you can search it with **SPL** (Search Processing Language), which works like Unix pipes. Each operator takes the output of the previous one.

```spl
index="botsv2" sourcetype="stream:http"
| stats count by src_ip
| sort -count
```

Read it like a recipe:

1. Look at every record in the botsv2 index that came from HTTP traffic.
2. Group by source IP and count how many records each one has.
3. Sort by count, biggest first.

That single query, with the right index and sourcetype, finds the **noisiest source IP** talking to any given host. Which is exactly how you find a vulnerability scanner. Knowing that pattern alone catches an entire class of attacks.

## The Single Most Useful First Query

I will tell every beginner this:

```spl
| metadata type=sourcetypes index=botsv2
| sort -lastTime
```

That is not a search. It is a *menu*. It returns every sourcetype in the index, sorted by when each one last got data. The first time I ran it, I saw `pan:traffic`, `stream:http`, `stream:smtp`, `stream:dns`, `stream:ftp`, `stream:tcp`, `osquery`, `XmlWinEventLog`, `WinRegistry`, and twenty more.

Knowing what is available before you start is the difference between targeted pivots and keyword spam. I spent the first hour of day one keyword-spamming. The next two and a half days were targeted pivots.

## The Pivoting Rhythm

Once you have the sourcetype menu, an investigation has a rhythm. Every step uses a value from the previous step.

Insider threat scenario, beat by beat:

1. **What is the user's IP?** Query *pan:traffic* with `src_user=` and the username. Get an IP.
2. **What sites did that IP visit?** Query *stream:http* with that IP. Get a list of unique domains.
3. **What did they download or upload to the suspicious site?** Filter the same query to that site, list URIs.
4. **What emails did the user send afterwards?** Query *stream:smtp* with the user's email address. Read sender, recipient, subject, attachments.

That is four queries. Each one needed exactly one value from the previous one. Once I saw that pattern, *everything in BOTSv2 felt the same*. Find a thread, follow it through the data sources, write down each pivot value, move on.

## Things That Clicked

**Curly braces in field names mean multi-value.** A field like `attach_filename{}` shows up that way because an email can have multiple attachments. The brace tells Splunk to return them all. I forgot the braces a few times early on and got blank results, then got annoyed at Splunk before realizing the issue was me.

**`src_ip` is the sender, `dest_ip` is the receiver.** Sounds obvious. Wasn't, the first time I saw a private IP in the `dest_ip` field. NAT explains it: the public-facing server has a public IP, but inside the network it is behind a router doing translation. So a request from the internet ends up with a private `dest_ip` after the network translates it.

**Private IPs only live in three blocks: `10.x.x.x`, `172.16.x.x` through `172.31.x.x`, and `192.168.x.x`.** Anything else is public. Once I memorized those, distinguishing internal traffic from external became automatic.

**Port 80 plus Apache plus a 200 OK means a web server.** Said out loud, this is "the obvious thing", but I had not put it together as a single recognizable pattern. Now I see it instantly.

**SQL injection looks like normal SQL plus one weird function.** *SELECT*, *FROM*, *WHERE* are everywhere. *UPDATEXML*, *SLEEP*, *BENCHMARK*, *EXTRACTVALUE* are the ones that mean someone is forcing the database to leak information through error messages or timing. Scanning a query for the *unusual* function is the actual skill.

**Cross-site scripting (XSS) shows up in requests containing both `<script>` and `cookie`.** That combination has almost no legitimate use. Adding a WAF rule for it would catch most amateur XSS overnight.

**Time correlation is a real superpower.** When a malware file got written, I narrowed the DNS query log to a ten-minute window around that timestamp and the C2 lookups jumped out like they were highlighted. Without the narrow window I had thousands of irrelevant DNS records.

**Dynamic DNS is a tell.** Malware uses providers like *duckdns.org*, *no-ip.org*, *hopto.org*, and *ddns.net* because the operator can change the resolved IP at any time without re-registering anything. Almost no business has a legitimate reason to use these in production. Alerting on DNS queries to them is a high-signal, low-cost detection.

## Things That Did Not Click For A While

**Base64 is everywhere and I kept skipping past it.** Twice during the 400 series the *answer to the next question* was sitting in a Base64 blob I had already seen and not bothered to decode. The lesson is to **decode Base64 immediately, every time**, the moment it appears. CyberChef's Magic operation handles recursive Base64 (Base64 inside Base64) in seconds.

**Reading PowerShell EncodedCommand is its own skill.** `powershell -enc <long-string>` is a textbook obfuscation pattern. The decoded payload is usually itself partially encoded, often with null bytes between characters because PowerShell expects UTF-16 little-endian. CyberChef's *From Base64 → Remove null bytes → Extract URLs* recipe is the combination I now use without thinking.

**Korean fonts on Kali don't render by default.** One artifact in the APT investigation was a Korean-language file, and the filename was just rendered as boxes in my terminal until I installed `fonts-noto-cjk`. The lesson generalizes: if a string looks like it should have characters but is rendering as boxes, the problem is your font, not your file.

**TryHackMe's answer form rejects raw Unicode.** I copy-pasted the Korean characters as the answer and it kept saying I was wrong. The fix was *CyberChef → Unescape Unicode Characters* before pasting. That is a TryHackMe-specific gotcha, but I want it written down somewhere because I would have given up otherwise.

**Document metadata can attribute attacks.** I had no idea this was a thing. The VirusTotal *Details* tab on an uploaded Office or PDF document shows the embedded *Author* and *Last Modified By* fields. Real attackers strip these. Lab documents and tutorial samples often don't, and the metadata tells you who built the file. This blew my mind for about an hour.

## The General Lessons I'm Keeping

- **Keyword search is the wrong first move.** Sourcetype enumeration is the right first move. Same idea on every new SIEM I ever touch.
- **Every investigation is a series of pivots, not a single query.** Get one value, pivot to a new data source with it, get the next value, repeat.
- **Time correlation rescues you when keyword search fails.** When you have a timestamp, narrow the window and look at everything in it.
- **Decode encodings immediately, do not skip them.** Base64, URL encoding, hex, Unicode escape. CyberChef handles all of them.
- **Anomaly is more useful than signature for blue team work.** Most attacker behavior shows up as the rare value in a field with low natural variety, not as a known-bad string.
- **VirusTotal is part of the investigation, not separate from it.** Hash gives detection names, metadata gives author, relations give network infrastructure. All three are pivots.

## What I'd Tell A Beginner

If you're about to start Splunk 2 or any long BOTS-style room, here is what I wish someone had told me on day one:

1. **Block three days, not three hours.** This is not a one-evening room. Plan for it.
2. **Start with the sourcetype menu, every single time.** Don't search until you have it.
3. **Write down every pivot value as you find it.** I kept a scratch text file with username, IP, hostname, email, hash, domain, filename, all stacked under section headers per scenario. Look back at it constantly.
4. **Install `fonts-noto-cjk` on Kali first.** Trust me.
5. **Keep CyberChef open in a tab.** You will use it more than you expect.
6. **Use the official BOTSv2 question list to guide your queries**, but don't read other people's answer walkthroughs. The walkthroughs are the answers without the methodology, and the methodology is the part that transfers.

This was the hardest lab I've done. It was also the one where the SOC analyst job clicked into focus as a real, learnable skill rather than a vague idea. If you're considering doing it, do it.

The technical walkthrough with the specific queries and screenshots lives in the [companion repo on GitHub](https://github.com/barrytd/security-lab-portfolio/tree/main/labs/2026-05-12-tryhackme-splunk-2). This post is the parts that generalize.
