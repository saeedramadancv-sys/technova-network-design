# 1. Requirements and Scope

## 1.1 Company profile

**TechNova Solutions** is a software and IT-services company with 320 employees
across three sites in Jordan.

| Site | Location | Role | Staff |
|---|---|---|---|
| HQ | Amman | Head office, data centre, all central services | 250 |
| BR1 | Irbid | Development branch | 45 |
| BR2 | Aqaba | Sales and support branch | 25 |

## 1.2 Business requirements

| ID | Requirement |
|---|---|
| BR-1 | All three sites must exchange internal traffic privately, without traversing the public Internet unencrypted. |
| BR-2 | All staff need Internet access; the company owns only one small public address block and cannot buy more. |
| BR-3 | Visitors and contractors need Internet access but must never reach internal systems. |
| BR-4 | The company web/application server must be reachable from the Internet. |
| BR-5 | A branch must keep working internally (addressing, printing, local file access) if its WAN circuit fails. |
| BR-6 | Departments must be logically separated so that a compromise in one does not trivially spread to others. |
| BR-7 | The design must leave room for roughly 50 % growth per site without re-addressing. |
| BR-8 | Network devices must be manageable remotely, but only from the IT management network. |

## 1.3 Technical requirements derived from the above

| ID | Requirement | Traces to |
|---|---|---|
| TR-1 | Private RFC 1918 addressing, hierarchically assigned with VLSM from one parent block. | BR-2, BR-7 |
| TR-2 | One contiguous address block per site, so each site can be advertised as a single summary route. | BR-1, BR-7 |
| TR-3 | Dynamic interior routing with sub-second-to-few-second reconvergence and no manual route maintenance. | BR-1 |
| TR-4 | VLAN segmentation per department, with inter-VLAN routing controlled at a single enforcement point per site. | BR-6 |
| TR-5 | NAT/PAT at the Internet edge; one static NAT translation for the public web server. | BR-2, BR-4 |
| TR-6 | A guest VLAN with an access-control policy of *Internet + DNS only*. | BR-3 |
| TR-7 | DHCP served locally at each branch, centrally at HQ. | BR-5 |
| TR-8 | Encrypted management (SSH v2) restricted by ACL to the management subnet; no Telnet. | BR-8 |
| TR-9 | Layer-2 hardening: port security, DHCP snooping, BPDU guard, unused ports disabled. | BR-6 |
| TR-10 | Redundant core-to-access links, aggregated so that both are forwarding. | BR-1 |

## 1.4 Host-count requirements (inputs to the addressing plan)

| Site | Segment | Current hosts | Planned (with growth) |
|---|---|---|---|
| HQ | Engineering | 118 | 126 |
| HQ | Sales | 54 | 62 |
| HQ | Guest Wi-Fi | 45 | 62 |
| HQ | Voice (IP phones) | 55 | 62 |
| HQ | Administration | 26 | 30 |
| HQ | IT / Servers | 9 | 14 |
| HQ | Device management | 11 | 14 |
| BR1 | Staff | 28 | 30 |
| BR1 | Guest Wi-Fi | 12 | 14 |
| BR1 | Device management | 5 | 6 |
| BR2 | Staff | 12 | 14 |
| BR2 | Guest Wi-Fi | 5 | 6 |
| BR2 | Device management | 4 | 6 |

The "planned" column is what the addressing plan is sized against; it is the
next usable-host boundary at or above the current count plus growth.

## 1.5 Constraints

- **Address space:** a single `172.16.0.0/16` internal block and an
  ISP-assigned `203.0.113.0/29` public block. No additional public addresses.
- **Protocol:** the interior routing protocol must be an open standard
  (vendor-independent), which rules out EIGRP.
- **Hardware:** the design must be implementable with the device models
  available in Cisco Packet Tracer (see `09-build-guide-packet-tracer.md`).
- **Serial WAN:** branch circuits are leased serial links at 2 Mbps.

## 1.6 Out of scope

Wireless controller design, IPv6 deployment, site-to-site VPN tunnels, BGP
peering, stateful firewalling, QoS policy beyond voice VLAN separation, and
physical cabling/rack layout. §8.2 of the final report explains what each of
these would add and in what order they should be tackled.
