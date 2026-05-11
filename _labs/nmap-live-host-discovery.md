---
title: "Nmap Live Host Discovery"
date: 2026-04-14
platform: TryHackMe
difficulty: Easy
category: offensive
summary: "ARP, ICMP, and TCP/UDP ping-scan techniques and when each one is the right tool, with hands-on practice on a multi-host target range."
---

## Overview

Nmap Live Host Discovery is a foundational TryHackMe room that walks through every method nmap supports for figuring out *which hosts on a network are alive*, before any port scan is ever run.

This is the part of the engagement that beginners often skip. You can't scan a host until you know it's there, and how you ask *"are you there?"* matters enormously because the wrong probe type produces silence on segments where the right one would have responded.

## Tools Used

- **nmap** with various host discovery flags.
- **tcpdump** or **Wireshark** to see what nmap actually sends on the wire.
- **arp-scan** as a Layer 2 alternative on the local segment.

## Methodology

**Step 1 - Understand the discovery layers.** Host discovery probes work at different layers of the network stack, and each layer has different reachability constraints:

- **ARP scan (Layer 2):** only works on the local Ethernet segment. The host has to be on the same broadcast domain. Cannot cross a router. Extremely reliable and very fast.
- **ICMP echo (Layer 3 / ping):** can cross routers but is commonly blocked at firewalls. Many organizations disable ICMP echo as a *"hide from casual scans"* measure.
- **TCP ping (`-PS`, `-PA`):** sends a TCP SYN or ACK to a specific port and watches the response. Firewalls that drop ICMP often allow TCP traffic to standard ports like 80 and 443.
- **UDP ping (`-PU`):** less common, useful when other probes are filtered. UDP scanning is also dramatically slower because of how the protocol responds to closed ports.

**Step 2 - Pick the right probe for the segment.**

```bash
# On the local segment (fastest, most reliable):
sudo nmap -sn -PR 192.168.1.0/24

# When you can route to the target but ICMP might be blocked:
sudo nmap -sn -PE -PS80,443 -PA80,443 10.0.0.0/24

# When you only have routed UDP visibility:
sudo nmap -sn -PU53,161 <range>
```

`-sn` is *"no port scan"*: only do host discovery, do not scan any ports on the alive hosts. `-PR` is ARP. `-PE` is ICMP echo. `-PS80,443` sends TCP SYN to those ports. `-PA80,443` sends TCP ACK to those ports.

**Step 3 - Understand the `--reason` flag.** When a host appears alive in the results, nmap can be asked *why* it thinks so. This is the difference between *"the host is up"* and *"the host responded to my ARP request"* (which tells you it's on the local segment) versus *"the host responded with TCP RST on port 80"* (which tells you the host is routable and a firewall is not filtering that probe).

```bash
sudo nmap -sn -PE -PS80 --reason <range>
```

**Step 4 - Privileged vs unprivileged behavior.** Several discovery techniques *require root* because they construct raw packets. Without root, nmap silently falls back to less effective methods. The simplest example: without root, `-PS80` cannot send a SYN; it has to do a full TCP connect, which is slower and produces different log entries on the target.

**Step 5 - Target specification expansion.** nmap accepts ranges in several forms:

- CIDR: `10.0.0.0/24`
- Hyphenated range: `10.0.0.1-50`
- Multiple targets: `10.0.0.1 10.0.0.2 10.0.0.3`
- File of targets: `nmap -iL targets.txt`
- Exclusions: `nmap 10.0.0.0/24 --exclude 10.0.0.1`

All of these expand to the same thing internally. Pick whichever is easiest to read in the scan log.

## Key Takeaways

- **You can't scan a host until you know it's there.** Host discovery is a real step, not a default.
- **The right probe depends on the network layer.** ARP for local segments, ICMP echo where it works, TCP SYN/ACK as a fallback, UDP as a last resort.
- **Privilege changes nmap's behavior.** Run as root for raw-packet probes, unprivileged as a last resort.
- **`--reason` is the diagnostic flag every analyst should default to.** Knowing *why* nmap thinks a host is up is half of triaging the results.

## What a Defender Should Do

- Block inbound ICMP echo at the perimeter if the business doesn't need it. It is a free reduction in casual scan visibility.
- Configure host firewalls to silently drop probes to closed ports, not RST them. Drop-by-default is harder to scan than RST-by-default.
- Segment networks so that an attacker who reaches one segment cannot ARP-scan the whole enterprise. Layer 2 is fast and reliable for attackers as well as defenders.
- Watch your DHCP and ARP logs for unexpected scanning patterns. arp-scan style sweeps generate distinctive ARP request volume.
- Internal-only services (database, admin panels, monitoring) should never respond on any probe from outside their dedicated segment.
