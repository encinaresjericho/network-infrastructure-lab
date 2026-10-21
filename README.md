# Network Infrastructure & Troubleshooting Lab

A hands-on Cisco Packet Tracer portfolio lab for practicing small-enterprise LAN design, VLAN segmentation, inter-VLAN routing, DHCP, NAT/PAT, basic access control, and structured troubleshooting.

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [Lab Architecture](#lab-architecture)
- [Environment](#environment)
- [Addressing Plan](#addressing-plan)
- [Implementation](#implementation)
- [Troubleshooting Runbook](#troubleshooting-runbook)
- [Validation Checklist](#validation-checklist)
- [Evidence and Screenshots](#evidence-and-screenshots)
- [Skills Demonstrated](#skills-demonstrated)
- [Safety and Limitations](#safety-and-limitations)
- [Repository Structure](#repository-structure)
- [References](#references)

## Overview

Build a segmented enterprise-style network in **Cisco Packet Tracer**. The core LAN uses one router and two switches to connect three VLANs. An optional simulated ISP router and test server are included so you can validate outbound NAT/PAT against a reachable destination instead of only entering NAT commands.

### Planned lab scope

- One main router (**R1**) providing router-on-a-stick inter-VLAN routing
- Two access switches (**SW1** and **SW2**)
- Three VLANs: IT, Staff, and Guest
- DHCP pools for all three VLANs
- NAT/PAT overload toward an optional simulated ISP
- An extended ACL that blocks Guest clients from initiating connections to the IT and Staff subnets
- Four troubleshooting exercises with evidence, root-cause notes, resolution, and post-fix validation

This is a learning lab rather than a production network.

## Objectives

1. Design and document a small segmented IPv4 network.
2. Create VLANs and assign end-device switch ports to the correct VLAN.
3. Configure 802.1Q trunks between the router and switches.
4. Configure router subinterfaces for inter-VLAN routing.
5. Configure DHCP pools and verify client leases.
6. Configure NAT overload for the three inside networks.
7. Apply a Guest VLAN ACL and verify the intended restrictions.
8. Diagnose and document at least four deliberately introduced faults.
9. Use show commands and end-device utilities to validate the network after changes.

## Lab Architecture

![Network Infrastructure & Troubleshooting Lab Architecture](assets/network-lab-architecture.png)

The required core network is R1 + SW1 + SW2. The ISP router and external test server form an optional simulated outside network used to test the default route and NAT/PAT.

### Device inventory

| Device | Suggested Packet Tracer device | Role |
|---|---|---|
| R1 | Cisco 2911 router | VLAN gateways, DHCP, ACL, NAT/PAT |
| SW1 | Cisco 2960 switch | First access switch and trunk aggregation |
| SW2 | Cisco 2960 switch | Second access switch |
| ISP | Cisco 2911 router (optional) | Simulated upstream network |
| EXT-SRV | Server-PT (optional but recommended) | Test destination and optional DNS/HTTP service |
| IT-01, STAFF-01, GUEST-01 | PC-PT | Clients connected to SW1 |
| IT-02, STAFF-02, GUEST-02 | PC-PT | Clients connected to SW2 |

### Physical/logical port map

| Link / device | Interface | Connects to | Purpose |
|---|---|---|---|
| R1 | G0/0 | SW1 G0/1 | 802.1Q trunk for VLANs 10, 20, 30 |
| R1 | G0/1 | ISP G0/0 | Outside/WAN link (optional) |
| SW1 | G0/2 | SW2 G0/1 | Inter-switch trunk |
| ISP | G0/1 | EXT-SRV | External test network (optional) |
| SW1 | Fa0/1, Fa0/2, Fa0/3 | IT-01, STAFF-01, GUEST-01 | Access ports in VLANs 10, 20, 30 |
| SW2 | Fa0/1, Fa0/2, Fa0/3 | IT-02, STAFF-02, GUEST-02 | Access ports in VLANs 10, 20, 30 |

Links use copper straight-through or automatic connections in Packet Tracer.

## Environment

- Cisco Packet Tracer
- Router and switch models that support the commands used in the templates
- At least six PC-PT end devices for the complete two-switch test
- Optional ISP router and Server-PT to validate NAT/PAT
- Basic familiarity with IPv4 addresses, subnet masks, VLANs, and the command-line interface

## Addressing Plan

Each user VLAN gets a separate /24 subnet. In a /24 network, the mask is `255.255.255.0`; the first three octets identify the subnet for these particular networks, while the last octet varies between usable hosts.

| Segment | VLAN | Network / mask | Default gateway | DHCP range (after exclusions) | Purpose |
|---|---:|---|---|---|---|
| IT | 10 | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.21`–`192.168.10.254` | IT test clients |
| Staff | 20 | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.21`–`192.168.20.254` | Staff test clients |
| Guest | 30 | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.21`–`192.168.30.254` | Guest clients with restricted access |
| R1–ISP link | — | `203.0.113.0/30` | R1 `.2`, ISP `.1` | Static | Simulated WAN link |
| External test network | — | `198.51.100.0/24` | `198.51.100.1` | Static | ISP-side server segment |
| EXT-SRV | — | `198.51.100.10/24` | `198.51.100.1` | Static | Test destination; optional DNS/HTTP |

The WAN and external addresses use documentation/example ranges and are meant for a closed simulation, not direct Internet connectivity. The DHCP DNS option points to `198.51.100.10`; the Server-PT DNS service can provide name resolution when configured with the appropriate record.

## Implementation

Use the step-by-step guide in [`docs/implementation-guide.md`](docs/implementation-guide.md) and the templates in the `configs/` folder.

1. Build and cable the topology.
2. Configure VLANs and access ports on SW1 and SW2.
3. Configure the trunks.
4. Configure R1 subinterfaces, gateways, and DHCP.
5. Configure the optional ISP/test server.
6. Add NAT/PAT and the Guest ACL.
7. Set the six PCs to DHCP.
8. Run the validation checklist.
9. Introduce each troubleshooting fault one at a time, diagnose it, fix it, and record the observed result.

## Troubleshooting Runbook

See [`docs/troubleshooting-runbook.md`](docs/troubleshooting-runbook.md) for full procedures.

| Case | Injected fault | Expected symptom | Main checks |
|---|---|---|---|
| NET-TRBL-01 | A Staff PC access port is assigned to the wrong VLAN | PC receives an address from the wrong subnet or reaches the wrong gateway | `show vlan brief`, `ipconfig` |
| NET-TRBL-02 | VLAN 30 is omitted from a trunk's allowed list | Guest traffic / DHCP fails across that trunk | `show interfaces trunk`, `show vlan brief` |
| NET-TRBL-03 | A DHCP pool has an incorrect default-router value | Client lease exists but remote-subnet connectivity fails | `show ip dhcp binding`, `ipconfig /all`, pool configuration |
| NET-TRBL-04 | NAT outside marking or Guest ACL application is incorrect | External test fails, or Guest isolation does not behave as intended | `show ip nat translations`, `show ip nat statistics`, `show access-lists`, interface config |

## Validation Checklist

- [ ] All devices are cabled according to the port map and links are up.
- [ ] VLANs 10, 20, and 30 exist on both switches.
- [ ] End-device ports are assigned to the intended access VLAN.
- [ ] Router-facing and inter-switch links carry VLANs 10, 20, and 30.
- [ ] R1's three subinterfaces are up and use the correct gateway addresses.
- [ ] All six PCs receive an address from the correct DHCP pool.
- [ ] Each PC can ping its own VLAN's default gateway.
- [ ] Same-VLAN clients on different switches can reach each other.
- [ ] Intended inter-VLAN connectivity works for IT and Staff clients.
- [ ] Guest clients cannot initiate connections to the IT and Staff subnets.
- [ ] Guest clients can reach the external test server if the optional WAN/server setup is configured.
- [ ] `show ip nat translations` displays translations after an inside client generates traffic toward the external test server.
- [ ] Four troubleshooting exercises are documented with evidence and a successful retest.

## Evidence and Screenshots

Suggested screenshot filenames and evidence:

| Filename | Evidence |
|---|---|
| `01-topology.png` | Complete topology with device labels and links |
| `02-vlan-configuration.png` | `show vlan brief` on both switches |
| `03-trunks.png` | `show interfaces trunk` on SW1 and SW2 |
| `04-router-subinterfaces.png` | `show ip interface brief` on R1 |
| `05-dhcp-leases.png` | DHCP leases and one PC's `ipconfig /all` |
| `06-intervlan-tests.png` | Successful permitted ping tests |
| `07-nat-validation.png` | `show ip nat translations` after external test |
| `08-guest-acl-test.png` | Guest-to-internal blocked test and permitted external test |
| `09-troubleshooting.png` | Before/after evidence for a troubleshooting fault |

## Skills Demonstrated

- IPv4 addressing and subnetting fundamentals
- VLAN creation and access-port assignment
- 802.1Q trunk configuration
- Router-on-a-stick inter-VLAN routing
- DHCP pool configuration and lease validation
- NAT overload / PAT concepts
- Extended ACL configuration and verification
- Cisco IOS troubleshooting commands
- Layer 1–3 fault isolation and root-cause documentation
- Network diagrams, validation records, and technical documentation

## Safety and Limitations

- This is a Packet Tracer practice lab, not a production deployment.
- Interface names, command support, and Packet Tracer behavior may vary by device model/version.
- The optional ISP router and test server simulate an outside network; this setup does not provide real Internet access by itself.

## Repository Structure

```text
network-infrastructure-troubleshooting-lab/
├── README.md
├── assets/
│   ├── network-lab-architecture.svg
│   └── network-lab-architecture.png
├── configs/
│   ├── R1-router-on-a-stick.txt
│   ├── SW1-core-switch.txt
│   ├── SW2-access-switch.txt
│   └── ISP-router-optional.txt
├── docs/
│   ├── implementation-guide.md
│   ├── lab-architecture.md
│   ├── troubleshooting-runbook.md
│   └── validation-checklist.md
└── screenshots/
    └── README.md
```

## References

- Cisco, [Configure Inter-VLAN Routing with the Use of an External Router](https://www.cisco.com/c/en/us/support/docs/lan-switching/inter-vlan-routing/14976-50.html)
- Cisco, [NAT Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/nat/nat-configuration-guide/nat.html)
- Cisco Networking Academy, [Packet Tracer](https://www.netacad.com/courses/packet-tracer)
