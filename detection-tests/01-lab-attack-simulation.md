# Detection Test 01: Attack Simulation from the Lab VLAN

**Status:** 📋 Planned. Results will be added after the test is run.

## Objective

Prove the detection stack works end to end. Run a realistic attack chain from the isolated Lab VLAN, from first scan to a persistence attempt, and confirm each layer catches what it's supposed to:

- **Security Onion** (Suricata + Zeek) sees the network side
- **Wazuh** sees the host side
- **The gateway firewall** keeps the attack contained to the Lab

Anything that should fire but doesn't is a gap to tune, and finding those gaps is the point of the test.

## Scope and safety

- Everything stays inside my own lab. Attacker and target are both VMs on the isolated Lab VLAN.
- The target is a deliberately vulnerable VM with the Wazuh agent installed.
- Snapshot the target before starting and revert it afterward.
- No traffic leaves the Lab except the containment checks in TC-06, which are expected to be blocked.

## Environment

| Role | What it is |
|------|------------|
| Attacker | Parrot OS VM on the Lab VLAN |
| Target | Deliberately vulnerable Linux VM on the Lab VLAN, Wazuh agent installed |
| Network sensor | Security Onion on the hypervisor, fed by the port mirror |
| Host monitoring | Wazuh server on the dedicated security node |
| Containment | Gateway firewall, default deny between VLANs |

## Test cases

| ID | Step | MITRE ATT&CK | Expected detection | Result |
|----|------|--------------|--------------------|--------|
| TC-01 | Host discovery sweep of the Lab subnet | T1018 Remote System Discovery | Security Onion: Zeek connection logs show one host probing many | Pending |
| TC-02 | Port and service scan of the target (Nmap `-sV`) | T1046 Network Service Discovery | Security Onion: Suricata scan alerts, Zeek logs full of rejected and half-open connections | Pending |
| TC-03 | SSH password guessing against the target with a small wordlist | T1110.001 Password Guessing | Wazuh: repeated authentication failures, then a brute-force alert. Security Onion: burst of SSH sessions in Zeek | Pending |
| TC-04 | Exploit a vulnerable service on the target with Metasploit | T1210 Exploitation of Remote Services | Security Onion: Suricata exploit signature, if one exists for the service. Wazuh: unexpected process or shell activity | Pending |
| TC-05 | From the shell, create a local user and change a watched file | T1136.001 Create Local Account | Wazuh: new-user alert and file integrity monitoring change on account files | Pending |
| TC-06 | From the target, try to reach the Trusted and Servers VLANs | T1021 Remote Services (lateral movement attempt) | Gateway: connections denied and logged. Nothing reaches the other VLANs | Pending |

## Evidence to capture

- Screenshot of each alert or log entry as it appears in Security Onion and Wazuh
- Gateway deny logs for TC-06
- A timeline: when each step ran, and when the first alert for it appeared
- Blur or crop IP addresses and hostnames before anything goes public

## Success criteria

- Every test case produces at least one detection in the layer it targets, or the miss is documented with a reason and a tuning fix
- TC-06 is fully blocked. Any successful connection out of the Lab is a failure and gets fixed first
- Detection times are recorded for each step

## Results

*Not run yet. This section will hold the real results: what fired, what didn't, screenshots, and what I tuned afterward.*

## Cleanup

- Revert the target VM to its pre-test snapshot
- Confirm no test accounts or tools remain on any system outside the Lab
- Note any rule or policy changes made because of the results
