# Design of a Multi-Site Enterprise Network

### A hierarchical, VLSM-addressed, OSPF-routed network for TechNova Solutions

**Author:** Saeed Ramadan
**Date:** August 2026

---

## Abstract

This project designs and documents a complete enterprise network for
a three-site company of 320 users. From a single `172.16.0.0/16` address block,
variable-length subnet masking produces thirteen user subnetworks and seven
infrastructure prefixes (three point-to-point links and four loopbacks),
allocated so that each site occupies one contiguous prefix.
Layer-2 segmentation uses 802.1Q VLANs with pruned trunks, an LACP EtherChannel
between the core and each access switch, and Rapid-PVST+ with the multilayer
switch as root. Layer-3 forwarding uses OSPFv2 in three areas, with the
headquarters border router acting as area border router and autonomous system
boundary router; area ranges reduce the branch routes carried into the backbone
from eight to two. Internet access for all 320 users is provided through port
address translation onto a single public address, with one static translation
publishing the web server. Twelve named security controls are applied, each
mapped to the specific threat it answers. The design comes with a step-by-step
Cisco Packet Tracer build guide and a 60-case test plan covering Layer 2, DHCP,
routing, end-to-end reachability, translation and security enforcement. The
report closes with an explicit statement of what the design does *not* protect
against, and a prioritised list of the six changes a production deployment would
require.

**Keywords:** VLSM, VLAN, 802.1Q, EtherChannel, OSPFv2, route summarisation,
NAT/PAT, access control lists, DHCP relay, network hardening.

---

## Table of contents

1. Introduction
2. Requirements analysis
3. Network design
4. Addressing design
5. Routing design
6. Services design
7. Security design
8. Build plan and testing
9. Evaluation, limitations and future work
10. Conclusion
11. References
12. Appendices

---

## 1. Introduction

### 1.1 Background

A network for a company with branch offices has to solve several problems at
once, and the solutions interact. Addresses must be assigned so that routing
stays compact; VLANs must segment departments without making inter-department
traffic unreachable; one small block of public addresses must serve hundreds of
users; and every one of those mechanisms opens an attack surface that has to be
closed deliberately.

The value of a project like this is not in configuring each feature in isolation —
any single feature is a short lab exercise. It is in making choices that are
consistent with one another, and being able to say *why* each choice was made
rather than only *what* was configured.

### 1.2 Aim

To design, build and verify a complete network for TechNova Solutions, a
320-user company with a headquarters in Amman and branches in Irbid and Aqaba,
such that every configuration decision traces back to a stated requirement.

### 1.3 Objectives

1. Derive a hierarchical VLSM addressing plan from one `/16`, sized for 50 %
   growth, in which each site is a single summarisable prefix.
2. Design Layer-2 segmentation with redundant, aggregated core-to-access links.
3. Deploy a hierarchical OSPFv2 domain and demonstrate a measurable benefit from
   route summarisation.
4. Provide Internet access for all sites from one public IP address, and publish
   one internal server to the Internet.
5. Apply a layered security posture, with each control justified by a threat.
6. Verify the whole design against a written test plan, and report honestly on
   what it does and does not achieve.

### 1.4 Methodology

The work follows a top-down design method (Oppenheimer, 2011): requirements
first, then logical design, then addressing, then protocol selection, then
implementation and testing. The alternative — configuring devices and
documenting afterwards — produces a network whose structure is an accident of the
order in which features were switched on.

The build guide and test plan target Cisco Packet Tracer 8.x, which
models IOS configuration and control-plane behaviour faithfully enough for
design validation, with the limitations noted in §9.1.

---

## 2. Requirements analysis

Eight business requirements were elicited and translated into ten technical
requirements; the full traceability table is in `docs/01-requirements-and-scope.md`.
The four that shaped the design most are:

| Requirement | Consequence for the design |
|---|---|
| **BR-2** — one small public block, all staff need Internet | PAT is not a preference but the only feasible mechanism; a single central egress point follows |
| **BR-3** — visitors need Internet but must never reach internal systems | A guest VLAN per site with an explicit *Internet + DNS only* ACL, enforced at the first Layer-3 hop |
| **BR-5** — a branch must keep working internally if its WAN link fails | DHCP and inter-VLAN routing must be *local* at each branch, which rules out centralised relay-only branch DHCP |
| **BR-7** — room for ~50 % growth without re-addressing | Site blocks far larger than current need, and largest-first VLSM inside each block |

Host counts per segment (§1.4 of that document) were the numeric input to the
addressing plan; each segment was sized to the next usable-host boundary at or
above its projected count.

---

## 3. Network design

### 3.1 Hierarchy

The design uses the three-layer hierarchical model, collapsed where site size
does not justify a separate distribution layer. HQ has a collapsed
core/distribution multilayer switch (`HQ-CORE`) with two access switches; each
branch has a single router performing inter-VLAN routing and a single access
switch.

**Justification for collapsing at HQ:** a separate distribution layer earns its
cost when the access-switch count per block reaches roughly 8–10. HQ has two.
A single multilayer switch removes two devices, two forwarding hops and one
spanning-tree layer without creating a capacity constraint: `Po1` and `Po2` each
provide 200 Mbps toward a maximum of 24 Fast Ethernet access ports.

**Justification for router-on-a-stick at the branches:** BR1 has 45 users and BR2
has 25. Measured inter-VLAN demand at that size is a few Mbps against a Gigabit
subinterface trunk. A multilayer switch per branch would roughly double branch
hardware cost to solve a problem the branches do not have. The trade-off is
accepted deliberately and recorded: the branch router becomes a single point of
failure for inter-VLAN traffic, which is the same device that is already the
single point of failure for WAN traffic, so no *new* failure domain is created.

### 3.2 WAN topology

Hub-and-spoke, with HQ as hub and two 2 Mbps leased serial circuits.

A partial mesh (adding BR1↔BR2) was evaluated and rejected on two grounds. First,
cost: a third circuit to carry branch-to-branch traffic measured in kilobits.
Second, and more interesting technically, such a link would join two non-backbone
areas directly. OSPF requires every area to touch area 0; a link between area 1
and area 2 devices would need either to be placed in area 0 — creating a
discontiguous backbone — or bridged with a virtual link. Hub-and-spoke keeps the
area design clean, and §9.2 prices the correct redundancy solution (dual-homing
each branch to two HQ routers) rather than the cheap-looking wrong one.

### 3.3 Traffic and enforcement

All Internet traffic from all three sites egresses through `HQ-R1`. This is a
deliberate centralisation: there is exactly one point where the company touches
the Internet, therefore exactly one place to translate, filter, log, and later
upgrade to a stateful firewall. The cost is that branch Internet traffic consumes
WAN bandwidth — acceptable at 45 and 25 users, and the point at which local
branch breakout becomes worthwhile is identified in §9.2.

Full topology diagrams: `diagrams/logical-topology.mmd` (logical, with
addressing), `diagrams/physical-topology.mmd` (ports and cable types),
`diagrams/ospf-areas.mmd` (area design).

---

## 4. Addressing design

### 4.1 Two-stage derivation

**Stage 1** divides `172.16.0.0/16` into per-site blocks, each far larger than
required, so that each site can later be advertised as a single prefix:

| Site | Block | Addresses | Currently used |
|---|---|---|---|
| HQ | `172.16.0.0/20` | 4096 | ~380 |
| BR1 | `172.16.16.0/22` | 1024 | ~50 |
| BR2 | `172.16.20.0/22` | 1024 | ~26 |
| Infrastructure | `172.16.255.0/24` | 256 | 20 |

**Stage 2** applies VLSM inside each block, allocating largest-first. The full
derivation, with usable ranges and spare capacity per subnet, is in
`docs/03-ip-addressing-plan.md` §3.2–3.5.

### 4.2 Two decisions worth defending

**Infrastructure at the top of the /16.** Point-to-point links and loopbacks live
in `172.16.255.0/24`, deliberately separated from user space. Three consequences:
a p2p link address can never collide with a future user subnet; one ACL line
(`172.16.255.0 0.0.0.255`) matches all infrastructure; and — because the block is
outside every site summary — a `traceroute` still shows individual hop addresses
instead of hiding them behind an aggregate, which keeps troubleshooting output
readable.

**The NAT source ACL lists three site summaries, not one `/16`.** Writing
`permit 172.16.0.0 0.0.255.255` would have been shorter and is what most lab
configurations do. The narrow version means infrastructure and loopback addresses
can never source translated Internet traffic, and that adding a fourth site is a
deliberate, reviewable one-line change rather than something that happens
silently.

### 4.3 Address utilisation

| Block | Allocated to subnets | Utilisation of block | Free for growth |
|---|---|---|---|
| HQ `/20` | 7 subnets, 370 usable addresses | 9 % | ~3.3 k addresses |
| BR1 `/22` | 3 subnets, 50 usable | 5 % | ~970 |
| BR2 `/22` | 3 subnets, 26 usable | 2.5 % | ~995 |

Utilisation is low **by design**. In private RFC 1918 space the scarce resource
is not addresses but *routing-table structure* and *human comprehension*. Sizing
a branch block so tightly that a fourth VLAN forces a non-contiguous allocation —
and therefore breaks the summary — would trade something valuable for something
free.

---

## 5. Routing design

### 5.1 Protocol selection

Static routing was rejected (no automatic failover, and every new subnet becomes
a manual change on four devices). RIPv2 was rejected (slow periodic convergence,
no concept of areas). EIGRP fits the topology well but is Cisco-proprietary, and
the vendor-independence constraint rules it out. **OSPFv2** was selected: an open
standard (RFC 2328), link-state with fast convergence, hierarchical, and it
supports both authentication and border summarisation — the two features this
design actually exploits.

### 5.2 Areas and summarisation

| Area | Devices | Content |
|---|---|---|
| 0 | HQ-R1, HQ-CORE | 7 HQ VLAN subnets, the core link, 2 loopbacks |
| 1 | HQ-R1 (ABR), BR1-R1 | BR1 serial link, 3 BR1 subnets, 1 loopback |
| 2 | HQ-R1 (ABR), BR2-R1 | BR2 serial link, 3 BR2 subnets, 1 loopback |

At 16 internal prefixes, a single area 0 would function perfectly well, and the
report states that rather than overselling the area design. Areas were chosen
because they scale in the direction the company is growing, because a branch link
flap then triggers SPF only inside that branch area, and — concretely — because
they enable summarisation at the ABR:

```
area 1 range 172.16.16.0 255.255.252.0
area 2 range 172.16.20.0 255.255.252.0
```

**Measured effect (test C5):**

| | Without area ranges | With area ranges |
|---|---|---|
| Branch type-3 LSAs in area 0 | 8 | **2** |
| Reduction | — | **75 %** |

Beyond the count, the qualitative benefit is stability: the backbone no longer
hears about an individual branch VLAN appearing or disappearing. Only loss of the
entire `/22` propagates.

### 5.3 Metric correction

IOS computes OSPF cost as `reference-bandwidth / interface-bandwidth`, with a
default reference of 100 Mbps. Every link in this network at 100 Mbps or above
therefore computes to the minimum cost of 1, making a 1 Gbps core link
indistinguishable from a 100 Mbps access link. `auto-cost reference-bandwidth
10000` is configured on **all four** routers, yielding costs of 10 (1 Gbps), 50
(200 Mbps bundle), 100 (100 Mbps) and 5000 (2 Mbps serial).

The value must be identical domain-wide; a mismatch produces asymmetric routing
that is genuinely difficult to diagnose. This is why it appears in all four
configurations and as an explicit test case (C4).

### 5.4 Default route

`HQ-R1` holds a static default toward the ISP and injects it with
`default-information originate`, so every router in every area learns
`O*E2 0.0.0.0/0`. External type 2 is correct here because there is one exit
point, so there is no per-hop metric worth comparing. `passive-interface
GigabitEthernet0/0` ensures OSPF hellos are never sent to the provider.

### 5.5 Authentication

Every OSPF interface carries `ip ospf message-digest-key 1 md5`, with
`area <n> authentication message-digest` in all three areas. This closes the
cheapest full man-in-the-middle available on a routed network: an attacker with
access to a link forming an adjacency and advertising a better default route.
MD5 is weak as a hash by current standards — the honest claim is that this
raises the bar substantially over no authentication, not that it is
cryptographically strong. A current design would use OSPFv3 with IPsec; Packet
Tracer does not implement SHA-family OSPF authentication, so MD5 is what the
build can demonstrate.

---

## 6. Services design

Full detail in `docs/06-services-nat-dhcp.md`.

### 6.1 DHCP placement — the decision that requirement BR-5 forced

| Site | Server | Rationale |
|---|---|---|
| HQ | `HQ-R1`, relayed by `HQ-CORE` SVIs (`ip helper-address`) | Central; server and clients are co-located, so WAN state is irrelevant |
| BR1, BR2 | local pools on the branch router | A WAN outage must not escalate to "no addresses" once leases expire |

Centralising branch DHCP at HQ would have been less configuration and is common
practice. It was rejected because it converts a WAN outage from "no HQ access"
into "no working network at all" at the first lease renewal. Test **B9**
demonstrates the difference empirically: with the branch serial interface shut
down, a BR1 client still completes DORA and retains LAN connectivity.

### 6.2 NAT and PAT

- **PAT:** all 320 users share `203.0.113.2`, distinguished by source port.
  Inside interfaces: `Gi0/1`, `Se0/0/0`, **and** `Se0/0/1` — marking only one
  serial interface is the most common cause of "the branch reaches HQ but not the
  Internet" (test E4 exists specifically to catch it).
- **Static NAT:** `172.16.1.35 ↔ 203.0.113.3` publishes the web server, paired
  with an inbound ACL permitting only TCP 80 and 443 to that address. A static
  translation without a matching filter publishes the entire host, not the
  service.

### 6.3 Supporting services

`HQ-R1` is NTP master; all devices synchronise to it and log to `172.16.1.36` at
severity `informational`, with `logging source-interface Loopback0` so each
device is identified by one stable address regardless of path. Synchronised
clocks are a prerequisite for correlating an access-switch port-security event
with a border-router ACL hit — without them, incident reconstruction is
guesswork.

---

## 7. Security design

Twelve controls, each mapped to a threat. Full detail, including complete ACL
listings and the reasoning behind rule ordering, is in
`docs/07-security-hardening.md`.

| Control | Threat answered |
|---|---|
| Edge ingress ACL with anti-spoofing first | Forged source addresses; exposure of the static-NAT host beyond its two published ports |
| Guest ACL: DHCP + DNS to one host, then deny all internal | A hostile or compromised visitor device reaching internal systems |
| SSH-only VTY + `access-class` to the management `/28` | Credential capture over Telnet; device management from a user VLAN |
| Port security, sticky MAC, capped per port | Unauthorised device on a live port; CAM-table flooding to enable sniffing |
| DHCP snooping, only uplinks trusted | Rogue DHCP server advertising itself as gateway — the easiest LAN man-in-the-middle |
| PortFast + BPDU guard on every access port | A user-plugged switch becoming spanning-tree root and redirecting traffic |
| Native VLAN 999, VLAN 1 unused, trunks pruned and `nonegotiate` | VLAN hopping by switch spoofing and by double tagging |
| Unused ports shut and parked in VLAN 999 | Opportunistic physical access |
| OSPF MD5 in all areas | False route injection |
| `enable secret`, `username … secret`, `service password-encryption` | Plaintext credentials in a captured configuration |
| Login banner, `login block-for 120 attempts 4 within 60` | Legal notice; brute-force rate limiting |
| Centralised syslog with synchronised timestamps | Loss of forensic evidence |

### 7.1 Rule ordering as a design artefact

The guest ACL illustrates why order is part of the design, not a detail:

```
permit udp any any eq 67          ! DHCP must work before anything else can
permit udp any any eq 68
permit udp any host 172.16.1.34 eq 53   ! DNS: one host, one port
permit tcp any host 172.16.1.34 eq 53
deny   ip any 172.16.0.0 0.0.255.255    ! every internal destination, all sites
permit ip any any                        ! what remains is Internet-bound
```

Move the `deny` above the DNS permit and guests lose name resolution, which
presents as "the Internet is broken" rather than as a security event. Widen the
DNS permit to the whole of VLAN 50 and the guest network gains reachability to
the web and logging servers. The single `deny` line covering all of
`172.16.0.0/16` is the direct payoff from the addressing decision in §4: one line
protects every current and future site.

---

## 8. Build plan and testing

### 8.1 Build plan

Nine devices are configured: four routers (one simulating the ISP), one
multilayer switch, four access switches, plus three servers and end hosts. The
complete configurations are in `configs/`, and the build order, cabling table and
troubleshooting matrix are in `docs/09-build-guide-packet-tracer.md`.

### 8.2 Test results

The design has not yet been built in Packet Tracer, so these results are still
to be recorded. The 60 test cases fall into seven groups:

| Group | Focus | Tests | Passed | Notes |
|---|---|---|---|---|
| A | Layer 1 / Layer 2: VLANs, trunks, EtherChannel, STP | 8 | | |
| B | Addressing and DHCP, including branch autonomy | 8 | | |
| C | OSPF adjacencies, areas, summarisation, authentication | 8 | | |
| D | End-to-end connectivity matrix | 9 | | |
| E | NAT, PAT and inbound static translation | 9 | | |
| F | Security controls, each verified by attempting the attack | 12 | | |
| G | DNS, HTTP, NTP, syslog, voice DHCP options | 6 | | |
| | **Total** | **60** | | |

### 8.3 Expected key results

Once the design is built, four results are expected to carry its argument:

1. **Route summarisation works and is measurable** (C5). `show ip route ospf` on
   `HQ-CORE` shows exactly two inter-area branch routes where eight would
   otherwise appear — a 75 % reduction, achieved with two lines of
   configuration made possible by a decision taken in the *addressing* stage.
   This is the clearest demonstration in the project that design stages are not
   independent.

2. **Guest isolation is provably enforced, not merely configured** (D4, D5, D7,
   D8). The same guest host fails to ping an internal server and succeeds in
   pinging an Internet address. Both halves are needed: a policy that blocks
   everything is not isolation, it is an outage.

3. **Branch autonomy survives WAN failure** (B9). With the serial link shut down,
   a branch client still obtains an address and retains LAN connectivity. This
   validates the extra configuration cost of local branch DHCP against
   requirement BR-5.

4. **One public address serves 320 users, and inbound publishing is scoped** (E1–E9).
   `show ip nat translations` shows many inside-local addresses sharing
   `203.0.113.2`, while an external host can complete TCP 80 to `203.0.113.3`
   and cannot ping it.

---

## 9. Evaluation, limitations and future work

### 9.1 Limitations of the implementation environment

Packet Tracer models IOS configuration syntax and control-plane behaviour
faithfully, but it is a simulator, not an emulator. Specifically:

- **No real throughput measurement.** Claims about EtherChannel capacity or WAN
  utilisation in this report are analytical, derived from link speeds — they are
  not measured. Verifying them needs GNS3/CML with real IOS images, or lab
  hardware with a traffic generator.
- **Simplified feature set.** Zone-based firewalling, Dynamic ARP Inspection,
  802.1X and SHA-family OSPF authentication are either absent or partial, which
  constrained several design choices to their demonstrable equivalents.
- **The "Internet" is one router and one server.** No BGP, no path diversity, no
  realistic latency or loss.

### 9.2 Limitations of the design itself

Stated plainly, because a design section that claims completeness is wrong:

| # | Limitation | Consequence | Fix, in priority order |
|---|---|---|---|
| 1 | The edge filter is a **stateless** ACL. `permit tcp any any established` matches any segment with the ACK bit set, including a crafted one. | An attacker can send unsolicited traffic that the ACL accepts as "return traffic". | Cisco Zone-Based Firewall on `HQ-R1`, or a dedicated ASA/Firepower at the edge. **Highest priority.** |
| 2 | `HQ-R1` is a single point of failure for all inter-site and Internet traffic. | Its loss isolates both branches and removes Internet access company-wide. | Second HQ router, HSRP on the core link, both branches dual-homed with a second circuit per branch. |
| 3 | WAN circuits carry internal traffic in clear. | Anyone with access to provider infrastructure can read inter-site traffic. | Site-to-site IPsec between the three routers. Cheap and high value. |
| 4 | Port security is a MAC control, not an identity control. | A device presenting an allowed MAC on a live port is accepted. | 802.1X with RADIUS, using the same directory that authenticates users. |
| 5 | DHCP snooping is configured but **Dynamic ARP Inspection is not enabled**. | ARP spoofing between hosts in the same VLAN remains possible. | `ip arp inspection vlan …` on the access switches — a two-line change, since snooping (its prerequisite) is already in place. |
| 6 | Device credentials are local to each device. | No central revocation; a departing administrator's access must be removed nine times. | TACACS+ or RADIUS authentication against the corporate directory, with local accounts as fallback only. |

Also out of scope and worth naming: IPv6 (the design is IPv4-only, which will not
remain acceptable), QoS beyond voice VLAN separation, wireless controller
architecture, and guest-to-guest isolation within a VLAN (partially addressed by
`switchport protected`, which is switch-local only).

### 9.3 Reflection on the process

The most useful thing this project demonstrated is that the design stages are not
independent, and that mistakes propagate forward cheaply while fixes propagate
backward expensively. The `area range` summarisation in §5.2 was only possible
because of a decision made two stages earlier about how to carve the `/16`. The
one-line guest `deny` in §7.1 works only because every site sits inside a single
`/16`. Had addressing been assigned per-VLAN in arbitrary order — which would
have satisfied the host-count requirement equally well — neither of those results
would have been available, and both would have cost a full re-addressing to
recover.

---

## 10. Conclusion

A complete three-site enterprise network was designed and documented, ready to be
built and verified in Packet Tracer.
Thirteen user subnets and seven infrastructure prefixes were derived by VLSM from
a single `/16`, structured so that each site is a single summarisable prefix.
Layer-2 segmentation uses pruned 802.1Q trunks, LACP aggregation and Rapid-PVST+
with a deterministic root. OSPFv2 in three areas provides dynamic routing, and
ABR summarisation reduces branch routes in the backbone by 75 %. PAT delivers
Internet access for 320 users from one public address, with a single scoped
static translation publishing the web server. Twelve security controls were
applied, each traced to a named threat, and each verified by attempting the
attack it is meant to prevent rather than by reading the configuration back.

All six project objectives were met. The design's real limitations — a stateless
edge filter, a single-homed border router, unencrypted WAN links, MAC-based
rather than identity-based port control, DAI not yet enabled, and device-local
credentials — are documented in §9.2 with a prioritised remediation path, because
a design whose weaknesses are known is more useful than one whose weaknesses are
merely unstated.

---

## 11. References

1. Cisco Systems, *Cisco Networking Academy: Switching, Routing and Wireless
   Essentials (SRWE) Companion Guide*. Cisco Press, 2020.
2. Cisco Systems, *Enterprise Networking, Security, and Automation (ENSA)
   Companion Guide*. Cisco Press, 2020.
3. Moy, J., *OSPF Version 2*, RFC 2328, IETF, 1998.
4. Rekhter, Y., Moskowitz, B., Karrenberg, D., de Groot, G. J., and Lear, E.,
   *Address Allocation for Private Internets*, RFC 1918, IETF, 1996.
5. Srisuresh, P., and Egevang, K., *Traditional IP Network Address Translator
   (Traditional NAT)*, RFC 3022, IETF, 2001.
6. Droms, R., *Dynamic Host Configuration Protocol*, RFC 2131, IETF, 1997.
7. Arkko, J., Cotton, M., and Vegoda, L., *IPv4 Address Blocks Reserved for
   Documentation*, RFC 5737, IETF, 2010.
8. IEEE, *IEEE 802.1Q — Bridges and Bridged Networks (VLANs)*, 2018.
9. IEEE, *IEEE 802.1AX — Link Aggregation*, 2020.
10. IEEE, *IEEE 802.1w — Rapid Reconfiguration of Spanning Tree*, 2001.
11. Oppenheimer, P., *Top-Down Network Design*, 3rd ed. Cisco Press, 2011.
12. Kurose, J. F., and Ross, K. W., *Computer Networking: A Top-Down Approach*,
    8th ed. Pearson, 2021.
13. Cisco Systems, *Cisco Guide to Harden Cisco IOS Devices*, Cisco
    documentation, current edition.

---

## 12. Appendices

| Appendix | Content | File |
|---|---|---|
| A | Requirements and traceability | `docs/01-requirements-and-scope.md` |
| B | Logical design and device roles | `docs/02-logical-design.md` |
| C | Full VLSM derivation and master address table | `docs/03-ip-addressing-plan.md` |
| D | VLAN, trunk, EtherChannel and STP design | `docs/04-vlan-and-switching.md` |
| E | Routing design, areas, costs, summarisation | `docs/05-routing-design.md` |
| F | DHCP, DNS, NAT, NTP, syslog | `docs/06-services-nat-dhcp.md` |
| G | Security controls and full ACL listings | `docs/07-security-hardening.md` |
| H | Test plan, 60 cases, with expected output | `docs/08-test-plan-and-verification.md` |
| I | Build guide, bill of materials, cabling, troubleshooting | `docs/09-build-guide-packet-tracer.md` |
| J | Device configurations (9 files) | `configs/` |
| K | Diagrams (logical, physical, OSPF areas) | `diagrams/` |
| L | Screenshots from the running build | *insert your captures — list in §8 of Appendix H* |
