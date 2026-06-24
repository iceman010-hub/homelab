# Home Lab

My personal security and automation lab. I use it to practice pentesting, network hardening, and self-hosting, and to run local AI without sending anything to the cloud.

It's a learning environment — nothing here touches my employer's systems or data. I don't publish IPs, ports, config files, or exact hardware models either; the details below are intentionally high-level, the same least-disclosure habit I'd apply to a production environment.

![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)
![IDS](https://img.shields.io/badge/IDS-Suricata-blue?style=flat-square)
![VPN](https://img.shields.io/badge/VPN-WireGuard-orange?style=flat-square)
![SIEM](https://img.shields.io/badge/SIEM-In%20Progress-yellow?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-lightgrey?style=flat-square)

---

## Contents

- [Network](#network)
- [Detection Stack](#detection-stack)
- [Hardware & Services](#hardware--services)
- [SOC Roadmap](#soc-roadmap)
- [Pentesting](#pentesting)
- [About](#about)

---

## Network

Everything's split into VLANs with a default-deny firewall. A segment only gets the access it actually needs, and the rest is blocked.

| VLAN | What's on it |
|------|--------------|
| Management | Device and infrastructure admin |
| Trusted | My main personal devices |
| Servers | Proxmox host, NAS, DNS, core services |
| Lab | Pentesting targets — isolated |
| IoT | Smart-home gear, internet-only |
| Media | Streaming devices |
| VPN | Remote-access clients |

A few choices worth calling out:

- The **Lab** can't reach my trusted or server networks. If I pop a box during testing, it stays contained.
- **IoT** devices get internet and nothing else. No talking to my real machines.
- Both Lab and IoT have their own WiFi SSIDs, so the separation holds at the wireless layer too, not just on the wire.
- The default **VLAN 1** (the "everything lands here if misconfigured" trap) is blocked from all internal subnets.

This maps cleanly to the **CIA triad**: segmentation and encrypted VPN access cover confidentiality, the firewall + Suricata + Pi-hole cover integrity, and spreading services across hardware keeps things available.

---

## Detection Stack

### What's running now

Suricata runs at the **edge**, built into the router. It inspects north-south traffic (internet in and out) against known signatures and blocks the obvious bad stuff before it reaches the internal network.

The limitation: it's an edge appliance. It catches what crosses the perimeter, but it can't see **east-west** traffic between my internal VMs, and it doesn't give me deep forensics or exportable logs to dig into after an alert. It tells me *something* happened, not the full story.

### What's planned

To close those gaps, I'm building an internal SOC on the Proxmox host, on two pillars — one watching the network, one watching the hosts:

| Pillar | Platform | What it covers |
|--------|----------|----------------|
| Network (NSM) | **Security Onion** — bundles Suricata + Zeek + Elastic SIEM | East-west traffic between VMs: signature alerts (Suricata), full connection metadata (Zeek), and a searchable dashboard (Elastic) |
| Host / endpoint | **Wazuh agents** | File changes, logins, process execution, privilege escalation inside each VM |

**Why both?** Security Onion watches the wire — it sees the scan, the protocol behavior, the file transfer. Wazuh watches inside the machines — it sees what actually happened on the host after the packet landed. Network visibility tells you someone knocked; host visibility tells you whether they got in and what they touched. Together they cover the full path of an attack, from the first packet to a changed file on a server.

---

## Hardware & Services

Exact models are intentionally left out. What matters is the role each piece plays.

**Proxmox host** — a quad-core Intel box with 64GB RAM, running the VMs: Ubuntu for services and testing, Parrot OS for the security toolkit. Storage is a 3-drive **ZFS RAIDZ1** pool (~2TB usable, single-drive fault tolerance, snapshots on) with a separate off-array backup target. This host is also where the SOC lives — 64GB gives it room to run Security Onion and the lab targets without choking.

**AI inference node** — a dedicated unified-memory machine running local LLMs, kept entirely off the cloud.

**Services Pi** — Pi-hole for network-wide DNS filtering and Home Assistant for smart-home automation.

**VPN Pi** — a dedicated WireGuard endpoint, kept to a single role so the router doesn't have to carry it.

**Networking** — a prosumer router/firewall with a built-in IDS, plus Wi-Fi 6 access points, custom firewall policies, and the VLAN setup above.

---

## SOC Roadmap

Built in phases on purpose — the whole point is to learn each integration as I go, not deploy a black box.

| # | Phase | Status |
|---|-------|--------|
| 1 | Deploy Security Onion on the Proxmox host (management + sniffing interfaces); confirm Suricata + Zeek + Elastic are pulling traffic | 🔲 Planned |
| 2 | Configure a Proxmox port mirror so the sensor sees east-west traffic between VMs | 🔲 Planned |
| 3 | Roll out Wazuh agents on the Linux/Windows VMs and confirm host logs reach the dashboard | 🔲 Planned |
| 4 | Export router edge logs into the SIEM for full perimeter-to-host coverage | 🔲 Planned |
| 5 | Run a simulated attack from an isolated Lab VM and confirm the full kill chain shows up end to end | 🔲 Planned |

---

## Pentesting

Closed lab, vulnerable VMs as targets, nothing leaves the network. Mostly Nmap, Metasploit, Burp Suite, and Wireshark, plus whatever else ships with Parrot.

---

## About

I'm an IT System Administrator and finished my AAS in Cybersecurity at Dallas College in May 2026. Find me on [LinkedIn](https://www.linkedin.com/in/noevalencia).

<!-- you read the source. respect. there's a little more in /.well-known/ -->
