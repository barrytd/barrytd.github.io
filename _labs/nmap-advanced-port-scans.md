---
title: "Nmap Advanced Port Scans - Stealth Scans and Evasion"
date: 2026-07-04
platform: TryHackMe
difficulty: Medium
category: offensive
summary: "Odd TCP flag combinations that slip past simple firewalls, plus spoofing, decoys, and the idle scan for hiding where a scan comes from."
---

The basic scans finish the handshake or half-open it. The advanced scans do something different: they send TCP packets with flag combinations a normal connection never uses, which lets them read a port's state from how a firewall reacts. The second half of the room is about hiding the source of the scan.

## Flag-based scans

**Null (-sN)** sets no flags. **FIN (-sF)** sets only FIN. **Xmas (-sX)** sets FIN, PSH, and URG, so the packet is lit up like a Christmas tree. All three rely on the same rule: a closed port replies with RST, and an open port stays silent. So a lack of reply means the port is open or filtered, and an RST means closed.

The catch is what kind of firewall sits in front. A stateless firewall only checks for the SYN flag to spot a connection attempt, so these odd-flag packets walk past it. A stateful firewall tracks connections and blocks anything that is not part of a valid one, which makes these scans useless against it. That is the whole reason to know more than one scan type.

## ACK and Window scans read the firewall

**ACK (-sA)** sends a packet with only ACK set. It cannot tell open from closed, but it maps the firewall. A port that replies with RST is unfiltered, meaning the firewall let the packet through. A port that stays silent is filtered. So the ACK scan answers a different question: not "what is open," but "what does the firewall allow."

## Hiding the source

Spoofing your address only works when you can see the return traffic, so on its own it is limited. The room covers a few ways to obscure the source anyway:

- -S SPOOFED_IP sends packets with a fake source IP.
- --spoof-mac fakes the MAC address.
- -D DECOY1,DECOY2,ME runs a decoy scan, mixing your real IP into a crowd of fake ones so the target cannot tell which is really scanning.
- -sI ZOMBIE_IP is the idle scan. It bounces the scan off an idle third host and reads that host's IP ID counter to infer the target's port states, so your address never touches the target.

## Packet-level tricks

-f fragments the packets into 8-byte pieces and -ff into 16-byte pieces, which can slip probes past filters that inspect whole packets. --source-port makes traffic look like it comes from a trusted port such as 53. --data-length pads packets to look less like a scan.

## The one flag worth keeping

--reason prints why nmap reached each conclusion. An open port shows syn-ack, because the target sent back a SYN/ACK. Seeing the evidence rather than the label is the fastest way to actually understand what a scan is doing.

## Key Takeaways

Stealth scans are situational: they beat stateless firewalls and fall flat against stateful ones. The value is knowing which scan fits which defense, and knowing that spoofing, decoys, and the idle scan exist for when the source needs to stay hidden.
