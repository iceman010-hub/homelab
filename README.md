# Home Lab

My personal security and automation lab. I use it to practice pentesting, network hardening, and self-hosting, and to run local AI without sending anything to the cloud.

It's a learning environment, so nothing here touches my employer's systems or data. I also don't publish IPs, ports, or config files — the network details below are described at a high level on purpose.

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
- The default VLAN 1 (the "everything lands here if misconfigured" trap) is blocked from all internal subnets.

This maps cleanly to the **CIA triad**: segmentation and encrypted VPN access cover confidentiality, the firewall + Suricata + Pi-hole cover integrity, and spreading services across hardware keeps things available.

## Hardware & services

**Proxmox** runs the VMs — Ubuntu for services and testing, Parrot OS for the security toolkit.

**NVIDIA DGX Spark** runs my local LLMs through Ollama. This is where the "demanding" AI work lives.

**Raspberry Pi 5** handles the heavier always-on jobs: the NAS and Suricata IDS.

**Raspberry Pi 4** runs the lighter stuff — Home Assistant, Pi-hole, and the WireGuard endpoint — so the router doesn't have to.

**Networking** is Alta Labs (router + AP6) with an eero Max 7, custom firewall policies, and the VLAN setup above.

## Pentesting

Closed lab, vulnerable VMs as targets, nothing leaves the network. Mostly Nmap, Metasploit, Burp Suite, and Wireshark, plus whatever else ships with Parrot.

## About

I'm an IT System Administrator at Techni-Tool and finished my AAS in Cybersecurity at Dallas College in May 2026. Find me on [LinkedIn](https://www.linkedin.com/in/noevalencia).

<!-- you read the source. respect. there's a little more in /.well-known/ -->
