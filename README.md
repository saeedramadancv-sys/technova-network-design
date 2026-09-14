# TechNova Solutions — Enterprise Network Design Project

**Course:** Computer Networks
**Deliverable:** Multi-site enterprise network design, implementation (Cisco Packet Tracer) and evaluation
**Design space:** `172.16.0.0/16` (private) + `203.0.113.0/29` (ISP-assigned public)

---

## 1. What this project is

A complete design-and-build package for the network of a fictional company,
**TechNova Solutions**, which operates a headquarters in Amman and two branch
offices (Irbid and Aqaba). The project covers the full engineering cycle:

requirements → logical design → addressing → switching → routing → services →
security → implementation → testing → evaluation.

Every configuration file in `configs/` is a complete, paste-ready Cisco IOS
configuration. Every address in `docs/03-ip-addressing-plan.md` is derived by
VLSM from a single block and is verified against the configurations.

## 2. Repository layout

```
Networking-Project/
├── README.md                         ← you are here
├── CHECKLIST.md                      ← submission / grading checklist
├── TechNova-Network-Design-Report.docx  ← THE SUBMITTABLE REPORT (Word)
├── TechNova-Network-Design-Report.pdf   ← same report, fixed formatting
├── docs/
│   ├── 01-requirements-and-scope.md  ← business + technical requirements
│   ├── 02-logical-design.md          ← topology, hierarchy, device roles
│   ├── 03-ip-addressing-plan.md      ← VLSM derivation + master address table
│   ├── 04-vlan-and-switching.md      ← VLANs, trunks, EtherChannel, STP
│   ├── 05-routing-design.md          ← multi-area OSPFv2, summarisation
│   ├── 06-services-nat-dhcp.md       ← DHCP, DNS, NAT/PAT, NTP, Syslog
│   ├── 07-security-hardening.md      ← ACLs, port security, SSH, DHCP snooping
│   ├── 08-test-plan-and-verification.md ← 60 test cases + expected output
│   ├── 09-build-guide-packet-tracer.md  ← step-by-step build order
│   └── 10-final-report.md            ← the academic report (submit this)
├── configs/                          ← 9 paste-ready IOS configurations
│   ├── ISP-R.txt      HQ-R1.txt      HQ-CORE.txt
│   ├── HQ-ACC1.txt    HQ-ACC2.txt
│   ├── BR1-R1.txt     BR1-SW1.txt
│   └── BR2-R1.txt     BR2-SW1.txt
├── diagrams/                         ← Mermaid source for all figures
│   ├── logical-topology.mmd
│   ├── physical-topology.mmd
│   └── ospf-areas.mmd
└── presentation/
    └── slides.md                     ← 14-slide defence deck
```

## 3. Design summary (one page)

| Layer | Decision | Rationale |
|---|---|---|
| Topology | Hub-and-spoke WAN, collapsed core at HQ | Matches a small enterprise with a single data centre; cheapest link count |
| Addressing | VLSM from `172.16.0.0/16`, one contiguous block per site | Enables per-site route summarisation and keeps ACLs short |
| Switching | 802.1Q trunks, LACP EtherChannel core↔access, Rapid-PVST+ | Link redundancy without spanning-tree convergence penalty |
| Inter-VLAN routing | SVIs on an L3 switch at HQ; router-on-a-stick at branches | HQ traffic volume justifies hardware switching; branches do not |
| Routing protocol | OSPFv2, areas 0 / 1 / 2, MD5 authenticated | Open standard, fast convergence, area ranges summarise branch routes |
| Internet edge | PAT for all users, one static NAT for the web server | Single public /29, must be conserved |
| Security | Layered: edge ACL, guest isolation, port security, DHCP snooping, SSH-only management | Defence in depth; each control has a named threat it answers |

**Device inventory:** 4 routers (1 ISP-simulating), 1 multilayer switch, 4 access
switches, 3 servers, end hosts and IP phones.

## 4. How to reproduce the build

1. Read `docs/09-build-guide-packet-tracer.md` — it gives the exact device
   models, modules and cable types, in the order they must be placed.
2. Place and cable the devices.
3. Paste each file from `configs/` into the matching device
   (`enable` → `configure terminal` → paste → `end` → `write memory`).
4. Work through `docs/08-test-plan-and-verification.md`. All 60 tests should
   pass; each one lists the command and the expected result.

## 5. Honest scope limits

These are stated up front rather than hidden, and are discussed in
§8 of the final report:

- The ISP and the "Internet" are simulated by one router and one server. No
  BGP, no real upstream.
- The edge filter is a **stateless** extended ACL. A production edge needs a
  stateful firewall (Cisco ZBF/ASA); this is named as future work, not claimed
  as done.
- Each branch has a single WAN circuit. Branch WAN redundancy is designed on
  paper (§8.2 of the report) but not built, to keep the topology inside the
  scope of one course project.
- Wireless is represented by access points on the guest VLAN; WLC-based
  wireless design is out of scope.
