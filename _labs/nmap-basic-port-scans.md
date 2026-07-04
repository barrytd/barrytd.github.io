---
title: "Nmap Basic Port Scans - Connect, SYN, and UDP"
date: 2026-07-04
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "The three scans that find running services on a host, how each one works at the packet level, and when to reach for it."
---

Port scanning is how you find the services running on a host. This room covers the three basic nmap scans and, more usefully, why each one works the way it does.

## The three scans

**TCP Connect scan (`-sT`)** completes the full three-way handshake: SYN, SYN/ACK, ACK. If the handshake finishes, the port is open. It needs no special privileges, so it is the fallback when you are not root. The downside is that finishing the handshake means the connection is logged by the service.

**TCP SYN scan (`-sS`)** is the common default. It sends a SYN, waits for the SYN/ACK that signals an open port, then sends a RST to tear the connection down before it completes. Because the handshake never finishes, it is faster and quieter than a connect scan. It needs root because it crafts raw packets.

**UDP scan (`-sU`)** covers the services TCP scans miss, like DNS, SNMP, and DHCP. UDP has no handshake, so nmap infers state from what comes back or does not. It is slow, but skipping it means missing a whole class of services.

## Port states

A port is not just open or closed. Nmap reports six states, and the extra ones come from firewalls sitting in the path. **Open** means a service is listening. **Closed** means nothing is listening but the port is reachable. **Filtered** means a firewall is blocking nmap from telling either way. The rest are combinations nmap cannot fully resolve. As a pentester, **open** is the state you care about, because it means a live service to interact with.

## Scope and speed

A few flags control how much you scan and how fast:

- `-p-` scans all 65535 ports, `-F` scans the top 100, `-p1-1023` scans a range.
- `-T0` through `-T5` set timing, slowest to fastest. `-T4` is a reasonable default.
- `--min-rate` and `--max-rate` cap the packets per second.

Speed is a tradeoff. Fast scans finish sooner but are louder and more likely to trip detection. Slow scans blend into normal traffic.

## Key Takeaways

Run `-sS` for TCP by default, fall back to `-sT` without root, and do not skip `-sU`. The scan tells you what is open. What you do with an open port is the next room.
