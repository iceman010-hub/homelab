# Build Your Own Secure Home Lab

This is how I built my home lab, written so you can build your own with security designed in from the start. Every step is tied back to the CIA triad (Confidentiality, Integrity, Availability), so you know what each piece protects and why it's there.

> This is a reference for a lab, not a production blueprint. Test changes carefully, and never scan or attack networks or devices you don't own. All names and values below are placeholders, not my real configuration.

## Before you start

### What you need (by role)

- A prosumer gateway that supports VLANs, firewall rules, and a built-in IDS
- A hypervisor host (mine is quad-core with 64GB RAM)
- Four drives for a ZFS pool, plus a separate backup target off the array
- A few single-board computers (SBCs): one with NVMe storage for log collection, one for remote access, and one for home automation, kept apart from the security gear
- Wi-Fi access points that support multiple SSIDs mapped to VLANs

### Design principles

- **Default deny.** Block everything, then allow only what a segment actually needs.
- **Least privilege.** Every device, user, and service gets the minimum access to do its job.
- **Segment by trust level.** Keep untrusted and high-risk devices away from anything that matters.
- **Log what matters.** If you can't see it, you can't investigate it.
- **Assume something gets compromised.** Design so one bad device can't pivot to the rest.

## 1. Plan your segments

Decide your segments before touching any hardware. I use 8 VLANs: 7 active segments plus the default VLAN 1, which stays blocked.

| Segment | Suggested trust level | What goes on it |
|---------|-----------------------|-----------------|
| Management | High | Admin interfaces for network gear and the hypervisor |
| Trusted | High | Your main personal devices |
| Servers | Medium | Hypervisor, DNS, monitoring, core services |
| Lab | Untrusted | Attack tools and vulnerable targets |
| IoT | Untrusted | Smart-home devices |
| Media | Low | Streaming devices |
| VPN | Medium | Remote-access clients |
| Default (VLAN 1) | None | Nothing. Blocked from all internal subnets |

Then write down the rules. These are the ones that matter most in my build:

| From | To | Rule |
|------|----|------|
| Lab | Trusted, Servers | Deny |
| IoT | Any internal segment | Deny (internet only) |
| VLAN 1 | Any internal segment | Deny |
| Anything not listed | Anything | Deny (default) |

Fill in the rest for your own needs, one allow rule at a time.

**CIA:** Confidentiality. A compromised or untrusted device can't reach data on the segments you care about.

- Name your segments and give each a placeholder range (for example `<LAB_SUBNET>`) before assigning real values
- Write down every allowed path between segments; anything not written down is denied
- Create the VLANs on the gateway and map them to switch ports and SSIDs

## 2. Gateway: firewall and edge IDS

The gateway routes between VLANs, enforces the rules from step 1, and runs Suricata inline at the edge. Know its limits: an edge appliance like mine can block or alert, but it doesn't keep deep forensic data, which is why steps 5 and 6 exist.

**CIA:** Confidentiality and Integrity. Default-deny rules keep segments apart, and the IDS blocks known-bad traffic before it gets inside.

- Set inter-VLAN traffic to deny by default, then add only the allow rules from your plan
- Block all traffic to and from VLAN 1
- Enable the IDS in blocking mode and review what it flags
- Keep the gateway's admin interface on the Management VLAN only, never exposed to the internet

## 3. DNS filtering with Pi-hole

Pi-hole blocks known malicious, phishing, and tracking domains for the whole network at the DNS level.

**CIA:** Integrity and Confidentiality. Devices can't resolve many known-bad domains, which cuts off common malware and phishing paths.

- Install Pi-hole on an always-on device in the Servers segment
- Point your DHCP scopes at Pi-hole as the DNS server
- Choose your upstream DNS and blocklists, and review the query log now and then

## 4. Hypervisor and storage

Proxmox VE runs the virtual machines. Storage is four drives in a ZFS RAIDZ2 pool, so any two drives can fail without losing data. ZFS also checksums everything, so silent corruption gets detected and repaired. Snapshots cover "undo my last change", and a backup target off the array covers "the whole pool died". For the full 3-2-1 rule (3 copies, 2 kinds of media, 1 offsite), add an offsite copy on top.

**CIA:** Availability through two-drive fault tolerance and backups, and Integrity through ZFS checksums and snapshots.

- Install Proxmox VE and keep its web interface on the Management VLAN
- Create a RAIDZ2 pool from the four drives
- Schedule automatic snapshots and backup jobs to the off-array target
- Test a restore at least once; a backup you've never restored is a guess

## 5. Network security monitoring: Security Onion

Security Onion runs as a VM on Proxmox and bundles Suricata (signature alerts), Zeek (connection metadata), and Elastic (search and dashboards). A port mirror on the Proxmox bridge copies VM traffic to it, so it sees east-west traffic that the edge IDS can't.

**CIA:** Integrity and Confidentiality. You can see lateral movement and investigate what actually happened on the wire.

- Create a Security Onion VM with separate management and sniffing interfaces
- Set up port mirroring on the Proxmox bridge and send the mirrored traffic to the sniffing interface
- Confirm Suricata alerts and Zeek logs are showing up before you trust it

## 6. Host monitoring: Wazuh

The Wazuh server and dashboard run on a dedicated SBC with NVMe storage, which handles log writes far better than a memory card. Wazuh agents on the VMs and PCs report logins, file changes, and process activity back to it.

**CIA:** Integrity through file integrity monitoring, and Confidentiality through login and privilege auditing.

- Install the Wazuh server and dashboard on the SBC
- Enroll agents on your VMs and PCs
- Turn on file integrity monitoring for sensitive paths and check that alerts arrive

## 7. Remote access with MFA: NetBird

NetBird is a WireGuard-based VPN that ties access to identity instead of a shared key. Users sign in through an identity provider that enforces MFA, and access policies decide exactly which devices each person can reach. It's Zero Trust applied to a home network.

A self-hosted NetBird setup needs:

- A domain name pointing at your server, with TLS certificates
- The NetBird management, signal, and relay services (run as containers; check the docs for ARM support on your hardware)
- A few ports reachable from the internet for web traffic and for peers to find each other (see the official docs for the current list)
- An identity provider for login and MFA (NetBird supports several, including self-hosted ones)

**CIA:** Confidentiality. Remote access requires a verified identity plus MFA, not just a key, and policies limit what each device can reach.

- Deploy the NetBird services and point your domain at them
- Connect your identity provider and require MFA for every user
- Write access policies with least privilege in mind, instead of allowing everything
- Place remote clients in the VPN segment so the firewall rules still apply

## 8. Isolated attack lab

The Lab VLAN holds your attack VM (I use Parrot OS) and deliberately vulnerable targets. It has its own SSID and can't reach Trusted or Servers, so anything you compromise stays contained.

**CIA:** Confidentiality and Availability. Compromised targets can't touch real data, and heavy scanning can't disrupt the rest of the network.

- Put the attack VM and target VMs on the Lab VLAN only
- Confirm the Lab can't reach Trusted or Servers before you start testing
- Snapshot targets so you can reset them after each exercise

## 9. Validate it

A monitoring stack you haven't tested is just an assumption. Attack your own lab and confirm each layer sees it.

**CIA:** Integrity. You prove the controls actually detect what they're supposed to.

- From the attack VM on the Lab VLAN, scan a target:

  ```bash
  nmap -sV <TARGET_IP>
  ```

- Run an exploit against a vulnerable target VM in the Lab
- Check that Security Onion flagged the network activity and Wazuh flagged the host activity
- Write down what fired, what didn't, and tune from there

## CIA triad summary

| | Controls in this build |
|---|---|
| **Confidentiality** | VLAN segmentation, default-deny firewall, NetBird with MFA and access policies, isolated Lab |
| **Integrity** | Edge IDS, Pi-hole DNS filtering, Security Onion, Wazuh file integrity monitoring, ZFS checksums and snapshots |
| **Availability** | ZFS RAIDZ2 (two-drive fault tolerance), off-array backups, services split across separate hardware |

## Common mistakes

- Leaving VLAN 1 usable, so misconfigured devices land somewhere with access
- Building a flat network where everything can talk to everything
- Keeping backups on the same array as the data they protect
- Exposing hypervisor or gateway admin pages to the internet
- Running an IDS that nobody ever reads or tunes
- Never testing a restore or an attack, then assuming both work
