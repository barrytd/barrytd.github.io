---
title: "Reconnaissance Fundamentals - Passive and Active Recon, and Where the Line Sits"
date: 2026-07-04
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "Recon splits into two modes: pulling public data without touching the target, and probing it directly. The difference decides whether you leave a trace."
---

Every engagement starts with the same question: what do I know about this target before I interact with it, and what can I only learn by interacting with it. That split is the whole of reconnaissance. Passive recon collects data from third parties without sending a single packet to the target. Active recon makes direct contact, which reveals more but leaves footprints. Knowing which mode you are in at any moment is the skill.

## Why the Distinction Matters

Passive recon is invisible to the target. You are querying registrars, DNS servers, certificate logs, and search engines, none of which belong to the organization you are looking at. Nothing shows up in their logs because you never talked to them.

Active recon is the opposite. The moment you ping a host, load its website, or connect to a port, you appear in access logs, firewall logs, WAF events, and IDS alerts. That is not a reason to avoid it, it is a reason to do it deliberately and, in a real engagement, only with signed authorization. Unauthorized probing is illegal in most places. On a lab network you own or a room scoped for you, this is a non-issue, but the habit of checking scope first is worth building early.

## Passive Recon: What Public Data Leaks

**WHOIS** and **RDAP** tell you who registered a domain and when. Personal details are almost always redacted now because of privacy laws, so the useful fields are the dates, the registrar, and the name servers. Registration and expiration dates let you estimate how long an organization has existed and when a renewal window opens, which is the kind of thing social engineering keys off of. WHOIS is being replaced by RDAP, which does the same job over HTTPS and returns structured JSON instead of freeform text.

One point that trips up beginners: the registrar and the name server are usually different companies. The registrar is who the domain was leased from. The name servers are whoever actually hosts the DNS, which is frequently a CDN like Cloudflare. Reading both correctly off a WHOIS record is a small thing that matters.

**DNS TXT records** are an underrated leak. A single lookup can expose an organization's entire vendor stack: which provider handles their email, what marketing platform they run, what SEO and analytics tools they use, and which services they have verified domain ownership with. None of that requires touching the target, and all of it maps out who to impersonate in a phishing scenario.

**Certificate Transparency logs** are the strongest passive tool for finding subdomains. Every TLS certificate issued by a participating authority is publicly logged, and each certificate lists the domains it covers. Searching those logs surfaces subdomains that a normal DNS lookup would never reveal, because you only resolve names you already know. Forgotten dev panels and staging sites show up here constantly.

**Shodan** indexes internet-facing services so you can study exposure at scale without scanning anything yourself. You can break down how many hosts run a given web server by country or by port, which is a useful way to understand what "normal" looks like before you go hunting for the abnormal.

## Active Recon: Making Contact

Once you decide to interact, a handful of simple tools carry most of the weight.

**ping** answers one question, is the host reachable, and gives you a bonus clue in the process. The TTL value in the reply hints at the operating system, because different systems start the TTL at different values. Linux typically starts at 64, Windows at 128. A reply with a TTL in the high 50s is usually a Linux box a few hops away, not a different OS. Plenty of hosts block ICMP entirely, so no reply does not mean the host is down.

**traceroute** maps the path between you and the target one router at a time. Counting hops is straightforward once you remember the last hop is the destination itself, not an intermediate router, so the routers between two systems is the total hop count minus one. When a trace goes quiet partway through and shows nothing but asterisks, that is usually a firewall or CDN that stops responding, which is itself a finding.

**Banner grabbing** with netcat, telnet, or curl is how you identify what software is running on an open port and what version. Many services announce themselves the moment you connect. FTP and SMTP hand you a banner immediately. For HTTP you request the headers and read the Server line. The version string is the payoff, because a known version can be matched against known vulnerabilities. curl -I for web services and nc for everything else cover most cases, and nc -v is worth remembering because verbose mode confirms the connection actually opened before the banner arrives.

## The Blue Team View

Everything above has a defender's mirror image. Organizations monitor their own certificate transparency entries to catch subdomains they did not authorize and dangling DNS records that invite takeover. They watch access logs and IDS alerts for the traceroute-and-banner-grab pattern that precedes a real attack. The defensive takeaways are boring and effective: redact WHOIS, keep unnecessary services off the public internet, and remember that anything in a public DNS record or a public certificate log is already known to anyone who looks.

## Key Takeaways

- Passive recon leaves no trace, active recon always does. Know which one you are doing at any given moment.
- The useful WHOIS fields are dates, registrar, and name servers. Do not confuse the registrar with the DNS host.
- Certificate Transparency logs are the best passive subdomain source. They reveal names a DNS lookup cannot.
- TTL is a free OS hint from a single ping. 64 leans Linux, 128 leans Windows, and hops lower the number as it travels.
- Banner grabbing turns an open port into a version string, and a version string is the start of everything that comes after.
- Do active recon only inside authorized scope. The tools are trivial, the legal line is not.
