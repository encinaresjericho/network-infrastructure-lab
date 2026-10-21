# Network Infrastructure & Troubleshooting Lab

*A Cisco Packet Tracer portfolio lab demonstrating small-enterprise LAN design, VLAN segmentation, inter-VLAN routing, DHCP, NAT/PAT, access control, and structured troubleshooting.*

## Project Overview

The lab models a small enterprise network with three departments—IT, Staff, and Guest—separated into VLANs. One router provides the VLAN gateways and routes traffic between VLANs. Two switches connect six client PCs. An optional simulated ISP router and external server provide a destination for testing NAT/PAT and Guest access rules.

### Project at a glance

| Area | Reference configuration |
|---|---|
| Simulator | Cisco Packet Tracer |
| Router | `R1` — Cisco 2911 |
| Switches | `SW1`, `SW2` — Cisco 2960 |
| User VLANs | VLAN 10 IT, VLAN 20 Staff, VLAN 30 Guest |
| Client PCs | `IT-01`, `STAFF-01`, `GUEST-01`, `IT-02`, `STAFF-02`, `GUEST-02` |
| Inter-VLAN routing | Router-on-a-stick using 802.1Q subinterfaces |
| Address assignment | DHCP pools on `R1` |
| Access control | Extended ACL blocks Guest-to-IT and Guest-to-Staff initiated traffic |
| External test | Optional `ISP` router and `EXT-SRV` server for NAT/PAT tests |
| Troubleshooting | Four fault-injection exercises with before/after checks |

### Skills demonstrated

- VLAN creation and access-port assignment
- 802.1Q trunk configuration
- Router-on-a-stick inter-VLAN routing
- DHCP scope configuration and lease checks
- NAT overload / PAT verification
- Extended ACL configuration and testing
- Cisco IOS `show` commands, PC IP checks, ping tests, and fault isolation

## Table of Contents

- [Lab Architecture](#lab-architecture)
- [Environment](#environment)
- [Addressing Plan](#addressing-plan)
- [Example Configuration](#example-configuration)
- [Implementation](#implementation)
- [Troubleshooting Runbook](#troubleshooting-runbook)
- [Validation Checklist](#validation-checklist)
- [Evidence and Screenshots](#evidence-and-screenshots)
- [Skills Demonstrated](#skills-demonstrated)
- [Lab Scope and Limitations](#lab-scope-and-limitations)
- [Repository Structure](#repository-structure)
- [References](#references)

## Lab Architecture

![Network Infrastructure & Troubleshooting Lab Architecture](assets/network-lab-architecture.png)

The required core network is `R1` + `SW1` + `SW2`. The optional outside network adds the `ISP` router and `EXT-SRV` so outbound NAT/PAT can be tested against a reachable destination.

```text
                         OPTIONAL SIMULATED OUTSIDE NETWORK

       EXT-SRV                         ISP
    198.51.100.10/24    ISP G0/1: 198.51.100.1/24
          |                            |
          +----------------------------+
                                       |
                              ISP G0/0: 203.0.113.1/30
                                       |
                               R1 G0/1: 203.0.113.2/30
                                  +----+----+
                                  |   R1    |
                                  | G0/0    |  Router-on-a-stick
                                  +----+----+
                                       | 802.1Q trunk (VLANs 10,20,30)
                                  SW1 G0/1
                              +--------+--------+
                              |       SW1       |---- G0/2 trunk ---- G0/1 ---- SW2
                              +---+-----+----+--+                                  +---+----+----+
                                  |     |    |                                    |   |    |
                                IT-01 STAFF-01 GUEST-01                          IT-02 STAFF-02 GUEST-02
                                VLAN10 VLAN20 VLAN30                            VLAN10 VLAN20 VLAN30
```

### Device inventory

| Device | Packet Tracer device | Main role |
|---|---|---|
| `R1` | Cisco 2911 | VLAN gateways, DHCP, Guest ACL, NAT/PAT |
| `SW1` | Cisco 2960 | First access switch and trunk connection to R1/SW2 |
| `SW2` | Cisco 2960 | Second access switch |
| `ISP` | Cisco 2911 | Simulated upstream router (optional) |
| `EXT-SRV` | Server-PT | External test host at `198.51.100.10` (optional) |
| `IT-01`, `STAFF-01`, `GUEST-01` | PC-PT | Clients connected to `SW1` |
| `IT-02`, `STAFF-02`, `GUEST-02` | PC-PT | Clients connected to `SW2` |

### Physical and logical port map

| Device / interface | Connects to | Purpose |
|---|---|---|
| `R1 G0/0` | `SW1 G0/1` | 802.1Q trunk carrying VLANs 10, 20, and 30 |
| `R1 G0/1` | `ISP G0/0` | Simulated WAN link (optional) |
| `SW1 G0/2` | `SW2 G0/1` | Inter-switch trunk carrying VLANs 10, 20, and 30 |
| `ISP G0/1` | `EXT-SRV` | External test network (optional) |
| `SW1 Fa0/1` | `IT-01` | VLAN 10 access port |
| `SW1 Fa0/2` | `STAFF-01` | VLAN 20 access port |
| `SW1 Fa0/3` | `GUEST-01` | VLAN 30 access port |
| `SW2 Fa0/1` | `IT-02` | VLAN 10 access port |
| `SW2 Fa0/2` | `STAFF-02` | VLAN 20 access port |
| `SW2 Fa0/3` | `GUEST-02` | VLAN 30 access port |

Use copper straight-through or Packet Tracer's automatic connection option for these Ethernet links.

## Environment

- Cisco Packet Tracer
- Cisco 2911 router models and Cisco 2960 switches
- Six PC-PT end devices
- Optional ISP router and Server-PT to test NAT/PAT and Guest access rules
- Familiarity with IPv4 addressing, subnet masks, VLANs, and Cisco IOS CLI

## Addressing Plan

Each user VLAN uses a separate `/24` subnet with mask `255.255.255.0`. The router subinterface is the default gateway for the VLAN. The first 20 addresses are excluded from DHCP, leaving `.21` through `.254` for client leases.

| Segment | VLAN | Network / mask | Default gateway | DHCP range | Purpose |
|---|---:|---|---|---|---|
| IT | 10 | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.21`–`192.168.10.254` | IT test clients |
| Staff | 20 | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.21`–`192.168.20.254` | Staff test clients |
| Guest | 30 | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.21`–`192.168.30.254` | Guest clients with restricted internal access |
| R1–ISP link | — | `203.0.113.0/30` | `R1: 203.0.113.2`, `ISP: 203.0.113.1` | Static | Simulated WAN link |
| External test network | — | `198.51.100.0/24` | `198.51.100.1` | Static | ISP-side server segment |
| `EXT-SRV` | — | `198.51.100.10/24` | `198.51.100.1` | Static | Optional test destination |

`203.0.113.0/24` and `198.51.100.0/24` are documentation/example IPv4 ranges. They are used here for a closed simulation, not as real Internet addresses.

### Example DHCP leases

The addresses below are plausible example leases, not fixed assignments. DHCP could assign another available address in the same range.

| PC | Switch / port | VLAN | Example IP | Mask | Gateway | DNS |
|---|---|---:|---|---|---|---|
| `IT-01` | `SW1 Fa0/1` | 10 | `192.168.10.21` | `255.255.255.0` | `192.168.10.1` | `198.51.100.10` |
| `STAFF-01` | `SW1 Fa0/2` | 20 | `192.168.20.21` | `255.255.255.0` | `192.168.20.1` | `198.51.100.10` |
| `GUEST-01` | `SW1 Fa0/3` | 30 | `192.168.30.21` | `255.255.255.0` | `192.168.30.1` | `198.51.100.10` |
| `IT-02` | `SW2 Fa0/1` | 10 | `192.168.10.22` | `255.255.255.0` | `192.168.10.1` | `198.51.100.10` |
| `STAFF-02` | `SW2 Fa0/2` | 20 | `192.168.20.22` | `255.255.255.0` | `192.168.20.1` | `198.51.100.10` |
| `GUEST-02` | `SW2 Fa0/3` | 30 | `192.168.30.22` | `255.255.255.0` | `192.168.30.1` | `198.51.100.10` |

The DNS option points to `EXT-SRV`. To test DNS lookups, enable the DNS service in Packet Tracer and create a record on the server. IP connectivity tests do not require DNS to be enabled.

## Example Configuration

The snippets below illustrate the main commands for this design. Interface names can differ by model, so verify the available interfaces in your Packet Tracer devices.

### SW1 — VLANs, access ports, and trunks

```text
enable
configure terminal
hostname SW1

vlan 10
 name IT
vlan 20
 name Staff
vlan 30
 name Guest

interface fa0/1
 description IT-01
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast

interface fa0/2
 description STAFF-01
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast

interface fa0/3
 description GUEST-01
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast

interface g0/1
 description TRUNK_TO_R1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30

interface g0/2
 description TRUNK_TO_SW2
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30

end
write memory
```

### SW2 — VLANs, access ports, and trunk

```text
enable
configure terminal
hostname SW2

vlan 10
 name IT
vlan 20
 name Staff
vlan 30
 name Guest

interface fa0/1
 description IT-02
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast

interface fa0/2
 description STAFF-02
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast

interface fa0/3
 description GUEST-02
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast

interface g0/1
 description TRUNK_TO_SW1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30

end
write memory
```

### R1 — Router-on-a-stick, DHCP, NAT/PAT, and Guest ACL

```text
enable
configure terminal
hostname R1

interface g0/0
 no shutdown

interface g0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 ip nat inside

interface g0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip nat inside

interface g0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip nat inside

interface g0/1
 description TO_ISP
 ip address 203.0.113.2 255.255.255.252
 ip nat outside
 no shutdown

ip dhcp excluded-address 192.168.10.1 192.168.10.20
ip dhcp excluded-address 192.168.20.1 192.168.20.20
ip dhcp excluded-address 192.168.30.1 192.168.30.20

ip dhcp pool VLAN10-IT
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 198.51.100.10

ip dhcp pool VLAN20-STAFF
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 198.51.100.10

ip dhcp pool VLAN30-GUEST
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 198.51.100.10

access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.20.0 0.0.0.255
access-list 1 permit 192.168.30.0 0.0.0.255
ip nat inside source list 1 interface g0/1 overload
ip route 0.0.0.0 0.0.0.0 203.0.113.1

ip access-list extended GUEST-IN
 permit udp any eq 68 any eq 67
 permit udp 192.168.30.0 0.0.0.255 host 198.51.100.10 eq domain
 permit tcp 192.168.30.0 0.0.0.255 host 198.51.100.10 eq www
 permit icmp 192.168.30.0 0.0.0.255 host 198.51.100.10 echo
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip 192.168.30.0 0.0.0.255 any

interface g0/0.30
 ip access-group GUEST-IN in

end
write memory
```

The Guest ACL permits DHCP, DNS/HTTP/ICMP to the external test server, denies Guest-initiated traffic to the IT and Staff subnets, and allows other Guest traffic after those denials. Verify ACL behavior using the exact Packet Tracer model and tests in the lab.

### ISP and external test server

Configure `ISP` as follows:

| Interface | IP address | Connected device / network |
|---|---|---|
| `G0/0` | `203.0.113.1/30` | `R1 G0/1` |
| `G0/1` | `198.51.100.1/24` | `EXT-SRV` network |

On `EXT-SRV`, set:

| Setting | Value |
|---|---|
| IPv4 address | `198.51.100.10` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `198.51.100.1` |
| DNS server | `198.51.100.10` (if running DNS service) |

Enable the server's HTTP and/or DNS service in Packet Tracer if those are part of the test. The external test network is simulated; it is not an Internet connection.

## Implementation

1. Build the topology and cable the devices using the port map.
2. Create VLANs 10, 20, and 30 on both switches.
3. Assign the six PC-facing ports to the intended access VLANs.
4. Configure the SW1-to-R1 and SW1-to-SW2 trunks to carry VLANs 10, 20, and 30.
5. Configure R1's three 802.1Q subinterfaces and gateway addresses.
6. Configure the DHCP exclusions and pools on R1.
7. If testing the outside network, configure ISP, EXT-SRV, the default route, NAT/PAT, and the external server's services.
8. Apply the Guest ACL to `R1 G0/0.30` and test both blocked and permitted traffic.
9. Set the six PCs to DHCP and inspect the assigned values.
10. Run the validation checklist and record the outputs.
11. Perform the four troubleshooting exercises one at a time; restore the working configuration after each case.

## Troubleshooting Runbook

The records below are worked examples of how to document faults. The symptoms, outputs, and resolutions are illustrative until reproduced in Packet Tracer.

| Case | Fault introduced | Expected symptom | Main checks |
|---|---|---|---|
| `NET-TRBL-01` | `STAFF-01` switch port is placed in VLAN 10 instead of VLAN 20 | Client obtains an IT-subnet lease or has incorrect gateway for its intended department | `show vlan brief`, `ipconfig /all` |
| `NET-TRBL-02` | VLAN 30 omitted from SW1–SW2 trunk allowed list | Guest connectivity/DHCP across the trunk fails for `GUEST-02` | `show interfaces trunk`, `show vlan brief` |
| `NET-TRBL-03` | VLAN 20 DHCP pool has wrong default-router value | Staff client receives a lease but remote-subnet connectivity fails | `show ip dhcp binding`, `ipconfig /all`, DHCP pool config |
| `NET-TRBL-04` | `R1 G0/1` is missing `ip nat outside` | Internal clients reach their gateway but cannot reach `EXT-SRV`; NAT translations are absent | `show ip nat translations`, `show ip nat statistics`, `show ip interface` |

### NET-TRBL-01 — Staff port assigned to the wrong VLAN

| Field | Example record |
|---|---|
| Issue | `STAFF-01` is connected to `SW1 Fa0/2` but the port has been assigned to VLAN 10 instead of VLAN 20. |
| Initial checks | On SW1, run `show vlan brief`. On STAFF-01, run `ipconfig /all`. |
| Example evidence | `Fa0/2` appears under VLAN 10; the client receives an address from `192.168.10.0/24`. |
| Root cause | The access port is configured for the wrong VLAN. |
| Resolution | On SW1, enter `interface fa0/2` then `switchport access vlan 20`. Renew the PC's DHCP lease. |
| Validation | `show vlan brief` lists `Fa0/2` in VLAN 20; `STAFF-01` receives a `192.168.20.x` address with gateway `192.168.20.1`; ping to its gateway succeeds. |
| Screenshot | `09-troubleshooting.png` — show the wrong VLAN, correction, and post-fix IP configuration. |

### NET-TRBL-02 — VLAN 30 omitted from the SW1–SW2 trunk

| Field | Example record |
|---|---|
| Issue | `GUEST-02` on SW2 cannot communicate as expected with the Guest VLAN because VLAN 30 is missing from the inter-switch trunk's allowed list. |
| Initial checks | On SW1 and SW2, run `show interfaces trunk` and `show vlan brief`. Inspect `SW1 G0/2` and `SW2 G0/1`. |
| Example evidence | The trunk is up, but VLAN 30 is not listed among the allowed VLANs on one side. |
| Root cause | VLAN 30 is not permitted on the inter-switch trunk. |
| Resolution | On the affected trunk interface, enter `switchport trunk allowed vlan 10,20,30`. |
| Validation | Both switches show VLANs 10, 20, and 30 allowed and active on the trunk; renew `GUEST-02`'s lease and test its gateway and Guest VLAN peers. |
| Screenshot | `09-troubleshooting.png` — capture the trunk output before and after restoring VLAN 30. |

### NET-TRBL-03 — Incorrect DHCP default-router option

| Field | Example record |
|---|---|
| Issue | `STAFF-02` receives an IP address but cannot reach other subnets. |
| Initial checks | Run `ipconfig /all` on STAFF-02 and `show ip dhcp binding` on R1. Inspect the `VLAN20-STAFF` pool. |
| Example evidence | The client has a `192.168.20.x` address but the DHCP-provided default gateway is incorrectly set to `192.168.10.1`. |
| Root cause | The Staff pool has the wrong `default-router` value. |
| Resolution | In `ip dhcp pool VLAN20-STAFF`, set `default-router 192.168.20.1`; renew the client lease. |
| Validation | `ipconfig /all` shows gateway `192.168.20.1`; the client can ping that gateway and reach permitted remote subnets. |
| Screenshot | `09-troubleshooting.png` — show the incorrect gateway, pool correction, and renewed lease. |

### NET-TRBL-04 — NAT outside interface not configured

| Field | Example record |
|---|---|
| Issue | An IT client can ping `192.168.10.1` but cannot reach the external test server at `198.51.100.10`. |
| Initial checks | Check the R1 WAN interface with `show ip interface brief`; test the WAN and server addresses; run `show ip nat statistics` and `show ip nat translations`. |
| Example evidence | `R1 G0/1` has `203.0.113.2` and is up, but it is not marked as the NAT outside interface; no translation appears after client traffic. |
| Root cause | The outside NAT role was not applied to `R1 G0/1`. |
| Resolution | Under `interface g0/1`, enter `ip nat outside`. Confirm `ip nat inside` is set on the three inside subinterfaces and the default route points to `203.0.113.1`. |
| Validation | Generate traffic from an inside client to `198.51.100.10`; run `show ip nat translations` and confirm a translation appears. Confirm the external test is reachable. |
| Screenshot | `07-nat-validation.png` or `09-troubleshooting.png` — capture NAT state after inside traffic and the successful retest. |

## Validation Checklist

Use this checklist while building the lab. A checkbox should be marked only after the corresponding result is observed in Packet Tracer.

- [ ] All devices are cabled according to the port map and links are up.
- [ ] VLANs 10, 20, and 30 exist on both switches.
- [ ] End-device ports are assigned to their intended access VLANs.
- [ ] Router-facing and inter-switch trunks carry VLANs 10, 20, and 30.
- [ ] R1's three subinterfaces are up and use the correct gateway addresses.
- [ ] All six PCs receive an address from the correct DHCP pool.
- [ ] Each PC can ping its own VLAN's default gateway.
- [ ] Same-VLAN clients on different switches can reach each other.
- [ ] IT and Staff clients can reach intended remote subnets.
- [ ] Guest clients cannot initiate connections to the IT and Staff subnets.
- [ ] Guest clients can reach `EXT-SRV` when the optional outside network is configured.
- [ ] `show ip nat translations` shows entries after an inside client generates traffic toward `EXT-SRV`.
- [ ] Four troubleshooting exercises have before/after evidence and a successful retest.

## Evidence and Screenshots

The screenshot filenames below give the project evidence a consistent order. Each image should show the Packet Tracer workspace, device CLI, or PC command window relevant to that check.

| Filename | What the screenshot should show |
|---|---|
| `01-topology.png` | Entire Packet Tracer workspace with R1, SW1, SW2, six PCs, device labels, links, and optional ISP/EXT-SRV topology |
| `02-vlan-configuration.png` | `show vlan brief` on SW1 and SW2 showing VLANs 10/20/30 and correct access-port membership |
| `03-trunks.png` | `show interfaces trunk` showing the router-facing and inter-switch trunks and VLANs allowed |
| `04-router-subinterfaces.png` | `show ip interface brief` on R1 showing G0/0.10, G0/0.20, G0/0.30 and G0/1 up/up |
| `05-dhcp-leases.png` | `show ip dhcp binding` plus a PC's `ipconfig /all` showing address, gateway, and DNS |
| `06-intervlan-tests.png` | Successful permitted ping tests, including same-VLAN clients connected to different switches |
| `07-nat-validation.png` | `show ip nat translations` after a client sends traffic to `198.51.100.10` |
| `08-guest-acl-test.png` | Guest-to-IT/Staff test failing as intended, and Guest-to-EXT-SRV test succeeding |
| `09-troubleshooting.png` | Before/after evidence from one fault, such as a wrong access VLAN, missing trunk VLAN, wrong DHCP gateway, or NAT outside setting |

### Examples of what successful evidence looks like

**VLAN check — illustrative CLI output**

```text
SW1# show vlan brief
VLAN Name                             Status    Ports
1    default                          active    Fa0/4, Fa0/5, ...
10   IT                               active    Fa0/1
20   Staff                            active    Fa0/2
30   Guest                            active    Fa0/3
```

**DHCP client check — illustrative PC output**

```text
PC> ipconfig
IP Address......................: 192.168.20.21
Subnet Mask.....................: 255.255.255.0
Default Gateway.................: 192.168.20.1
DNS Server......................: 198.51.100.10
```

**NAT check — illustrative router output**

```text
R1# show ip nat translations
Pro  Inside global       Inside local        Outside local      Outside global
icmp 203.0.113.2:1       192.168.10.21:1     198.51.100.10:1    198.51.100.10:1
```

The exact formatting and dynamic port/identifier values may differ by Packet Tracer version and traffic type. The important result is that the relevant VLAN, IP settings, ACL behavior, or NAT translation is visible.

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

## Lab Scope and Limitations

- This is a Cisco Packet Tracer practice lab rather than a production deployment.
- Interface names, command support, and Packet Tracer behavior can vary by device model and version.
- The optional ISP router and external server simulate an outside network; the design does not provide real Internet access by itself.
- Sample IP configurations, troubleshooting records, and CLI output in this README illustrate the expected format and should be matched to the actual saved Packet Tracer project.

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
