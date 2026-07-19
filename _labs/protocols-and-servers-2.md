---
title: "Protocols and Servers 2 - Sniffing, MITM, and Password Attacks on Cleartext Services"
date: 2026-07-04
platform: TryHackMe
difficulty: Medium
category: offensive
summary: "Three attacks against network protocols, sniffing, man-in-the-middle, and password attacks, and the encryption and authentication that shut each one down."
---

The first Protocols and Servers room showed that FTP, SMTP, POP3, and IMAP send everything in plain text. This follow-up is about what an attacker does with that fact. Three attacks come out of cleartext traffic and weak authentication: sniffing, man-in-the-middle, and password attacks. Each one maps to a specific defense, and understanding both sides is the point.

## The Framework: CIA and DAD

Two acronyms make the rest of this easier to reason about. **CIA** is what you protect: Confidentiality, Integrity, and Availability. **DAD** is what an attack causes: Disclosure, Alteration, and Destruction. They line up directly.

- Sniffing breaks **Confidentiality** and causes **Disclosure**. The attacker reads data they should not see.
- MITM breaks **Integrity** and causes **Alteration**. The attacker changes data in transit.
- Password attacks lead to **Disclosure** too, by handing over an account.

Different organizations weight these differently. An intelligence agency cares most about confidentiality, a bank cares most about the integrity of transactions, and an ad-supported platform cares most about availability. Knowing which one you are protecting tells you which attack matters most.

## Sniffing: Reading Traffic Off the Wire

A sniffing attack is network packet capture. When a protocol communicates in cleartext, anyone positioned to see the traffic can read the contents, including credentials. tcpdump and Wireshark are the standard tools. A filter as simple as a port number or a protocol name isolates the interesting traffic, and an ASCII view turns a POP3 or FTP login into a readable USER and PASS right in the output.

The common assumption is that TLS killed sniffing. It did not, it just narrowed where it works. Sniffing is still effective on internal corporate networks that never encrypted internal traffic, on legacy systems and IoT devices that only speak cleartext, on misconfigured services where TLS is available but not enforced, on wireless networks within range of the attacker, and after a MITM that stripped the encryption. On internal penetration tests, sniffing remains one of the most reliable ways to collect credentials.

The requirement for the attack is access to the traffic. That comes from a network tap, a switch configured for port mirroring, ARP spoofing on the local segment, or simply a compromised host on the same network.

## MITM: Sitting Between Two Parties

A man-in-the-middle attack puts the attacker in the middle of a conversation that both sides believe is private. The victim thinks they are talking to the server, the server thinks it is talking to the victim, and the attacker relays and optionally alters everything in between.

Getting into that position means redirecting traffic. **ARP spoofing** works on a local network by forging ARP replies so the victim's traffic for the gateway comes to the attacker instead. **DNS spoofing** hands out false DNS answers to send victims to attacker-controlled servers. **Rogue access points** are fake Wi-Fi networks named to look legitimate, so anyone who connects routes all their traffic through the attacker. **BGP hijacking** does the same thing at internet routing scale, which is a nation-state-tier attack rather than a LAN one.

Encryption complicates MITM but does not automatically stop it. **SSL stripping** downgrades a victim's HTTPS connection to HTTP, keeping a real HTTPS link to the server while serving the victim cleartext, and a user who never typed https:// may not notice the missing padlock. **Fake certificates** work if the victim clicks through the warning. A **compromised or rogue certificate authority** is the serious case, because it lets an attacker mint valid-looking certificates for any domain.

The defenses are worth naming because they explain why MITM is hard now. **HSTS** tells a browser to only ever use HTTPS for a domain, which defeats SSL stripping. **Certificate Transparency** logs every issued certificate publicly, so fraudulent ones get noticed. **Certificate pinning** hard-codes which certificate an app will accept, so even a compromised CA does not help. **DANE** publishes certificate info in DNS using DNSSEC as an alternative trust path. None of these matter if the underlying protocol is cleartext, which is the whole reason cleartext protocols are the root problem.

## Password Attacks

Even encrypted services fall to weak passwords. **Hydra** is the tool for testing this: point it at a service with a username and a wordlist and it tries combinations until one works. The mechanics are simple, a single login with -l, a wordlist with -P, and the service name, but the concept generalizes.

Modern password attacks go beyond plain brute force. **Credential stuffing** replays username and password pairs leaked from other breaches, betting on password reuse. **Password spraying** flips the loop, trying a few common passwords across many accounts to avoid lockouts. Both lean on the reality that people reuse passwords and pick weak ones.

The defenses are boring and effective: strong password policies with breached-password detection, account lockout or rate limiting on every authentication endpoint, and multi-factor authentication on anything sensitive. Where possible, passwordless options like passkeys or certificate-based authentication remove the guessable secret entirely.

## The Fix Is Encryption Plus Authentication

Every attack here traces back to two gaps: traffic sent in the clear, and authentication that can be guessed or replayed. The fixes follow. For confidentiality and integrity, wrap the protocol in TLS or replace it with a secure equivalent: HTTPS over HTTP, SSH over Telnet, SFTP or FTPS over FTP, and IMAPS, POP3S, and SMTPS over their cleartext versions. For authentication, use SSH keys instead of passwords, enforce MFA, and rate-limit logins.

One detail worth keeping straight is **implicit TLS versus STARTTLS**. Implicit TLS encrypts from the moment the connection opens, on a dedicated port like 993 or 995. STARTTLS starts unencrypted on the normal port and upgrades to TLS after the client asks, which is common on port 587 for mail submission. The upgrade path is convenient but means the connection is briefly cleartext, and a MITM can strip the upgrade if it is not enforced.

## Defensive Checklist

The room ends with a checklist that doubles as a decent audit baseline: all services on TLS 1.2 or 1.3 with strong ciphers, cleartext protocols disabled or isolated, SSH set to key-based auth with passwords off, strong password policy with breach detection, lockout or rate limiting everywhere, MFA on sensitive systems, network segmentation to contain sniffing, proper certificate validation, HSTS on web apps, and logging that catches authentication anomalies.

## Key Takeaways

- **CIA is what you defend, DAD is what the attack causes.** Sniffing to Disclosure, MITM to Alteration, password attacks to Disclosure.
- **Sniffing is not dead, it moved indoors.** Internal networks, legacy systems, and IoT still hand credentials to anyone on the segment.
- **MITM needs a position in the traffic.** ARP spoofing, DNS spoofing, and rogue access points are the common ways to get it.
- **HSTS, Certificate Transparency, and pinning are why MITM got hard,** but none of them help a protocol that never encrypts.
- **Password attacks evolved into stuffing and spraying.** MFA and rate limiting matter more than password complexity alone.
- **The fix is always encryption plus real authentication.** TLS and SSH close the cleartext gap, keys and MFA close the credential gap.
