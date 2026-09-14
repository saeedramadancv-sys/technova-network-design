# Presentation deck — 14 slides

Speaker notes are the indented lines. Target: 12–15 minutes plus questions.
Each slide has one message; if you cannot say the message in one sentence, the
slide is doing too much.

---

## Slide 1 — Title

**Design and Implementation of a Multi-Site Enterprise Network**
TechNova Solutions — HQ Amman, branches Irbid and Aqaba
Your name · Student ID · Course · Date

> One sentence: "320 users, three sites, one /16 and one public IP address —
> here is how it fits together."

---

## Slide 2 — The problem

- 320 users, 3 sites, 13 network segments
- **One** public IP block (`203.0.113.0/29`) for everyone
- Visitors need Internet, must never touch internal systems
- A branch must keep working when its WAN link dies
- Room for 50 % growth without re-addressing

> Do not list all 8 business requirements. These five are the ones that changed
> the design; the rest are in the report.

---

## Slide 3 — Topology

*Figure: `diagrams/logical-topology.mmd`*

- Collapsed core at HQ, router-on-a-stick at branches
- Hub-and-spoke WAN, 2 Mbps serial
- One Internet egress point, at HQ

> If asked why not a full mesh: a third circuit for kilobits of branch-to-branch
> traffic, and it would put a link between two non-backbone OSPF areas. Say that
> and stop — it is a complete answer.

---

## Slide 4 — Addressing: two stages, not one

**Stage 1 — site blocks**

| HQ | BR1 | BR2 | Infrastructure |
|---|---|---|---|
| `172.16.0.0/20` | `172.16.16.0/22` | `172.16.20.0/22` | `172.16.255.0/24` |

**Stage 2 — VLSM inside each block, largest first**

> The message: sites got contiguous blocks *before* any VLAN got an address.
> That ordering is what makes slide 7 possible.

---

## Slide 5 — HQ VLSM derivation

| Segment | Hosts | Prefix | Subnet |
|---|---|---|---|
| Engineering | 126 | /25 | 172.16.0.0/25 |
| Sales | 62 | /26 | 172.16.0.128/26 |
| Guest | 62 | /26 | 172.16.0.192/26 |
| Voice | 62 | /26 | 172.16.2.0/26 |
| Admin | 30 | /27 | 172.16.1.0/27 |
| Servers | 14 | /28 | 172.16.1.32/28 |
| Management | 14 | /28 | 172.16.1.48/28 |

> Largest first. If asked why: allocating in arbitrary order fragments the block
> and eventually forces a non-contiguous allocation, which breaks summarisation.

---

## Slide 6 — Layer 2

- 802.1Q trunks with **pruned** allowed-VLAN lists
- LACP EtherChannel core↔access: **both** links forward
- Rapid-PVST+, HQ-CORE is root for every VLAN
- Native VLAN moved to **999**, VLAN 1 unused everywhere

> The EtherChannel point in one line: with plain redundant links STP blocks one
> of them, so you pay for two and use one.

---

## Slide 7 — OSPF areas and the summarisation result

*Figure: `diagrams/ospf-areas.mmd`*

```
area 1 range 172.16.16.0 255.255.252.0
area 2 range 172.16.20.0 255.255.252.0
```

| Branch routes in the backbone | Without | With |
|---|---|---|
| | 8 | **2** |

> **This is the slide that wins marks.** Two lines of configuration, 75 % fewer
> routes — and it was only possible because of the decision on slide 4. Say that
> connection out loud.

---

## Slide 8 — One number worth fixing

Default OSPF reference bandwidth = 100 Mbps →
every link ≥ 100 Mbps costs **1**. A 1 Gbps link and a 100 Mbps link look
identical to SPF.

```
auto-cost reference-bandwidth 10000   ! on ALL four routers
```

> If asked what happens when it is set on only some routers: asymmetric routing,
> and it is genuinely hard to diagnose. That is why it is a test case.

---

## Slide 9 — NAT: 320 users, one address

- PAT: everyone shares `203.0.113.2`, distinguished by source port
- Static NAT: `172.16.1.35 ↔ 203.0.113.3` publishes the web server
- `ip nat inside` on `Gi0/1` **and both** serial interfaces

> Mention the both-serials point. It is the classic failure: branch pings HQ
> fine, cannot reach the Internet, and the cause is one missing line.

---

## Slide 10 — Guest isolation, and why order matters

```
permit udp any any eq 67 / 68            ← DHCP first
permit udp any host 172.16.1.34 eq 53    ← DNS: one host, one port
deny   ip any 172.16.0.0 0.0.255.255     ← all internal, all sites, ONE line
permit ip any any                        ← the rest is the Internet
```

> Move the deny up one line and guests lose DNS — which they will report as
> "the Internet is broken", not as a security event. The single deny line
> covering every site is the payoff from slide 4.

---

## Slide 11 — Security: 12 controls, 12 threats

| Control | Threat |
|---|---|
| Edge ACL, anti-spoofing first | Forged sources, over-exposed NAT host |
| Guest ACL | Hostile visitor device |
| SSH only + mgmt ACL | Credential capture, management from user VLANs |
| Port security (sticky) | Rogue device, CAM flooding |
| DHCP snooping | Rogue DHCP = easiest MITM |
| BPDU guard | User switch becoming STP root |
| Native VLAN 999 | VLAN hopping (double tagging) |
| OSPF MD5 | False route injection |

> Every control names a threat. If you cannot name the threat, do not put the
> control on the slide.

---

## Slide 12 — Results

- **60 tests**, 7 groups, all passing
- Summarisation: 8 branch routes → **2** *(screenshot)*
- Guest: server ping **fails**, `192.0.2.53` ping **succeeds** *(screenshot)*
- Branch WAN shut down → branch DHCP **still works** *(screenshot)*
- External host loads `http://203.0.113.3` *(screenshot)*

> Show the screenshots, do not describe them. Two of them, not five — pick
> summarisation and guest isolation.

---

## Slide 13 — What this design does *not* do

1. Edge filter is **stateless** — needs a real firewall (ZBF / ASA)
2. `HQ-R1` is a single point of failure
3. WAN links are unencrypted — needs IPsec
4. Port security is MAC-based, not identity-based — needs 802.1X
5. DHCP snooping is on, **DAI is not** — two-line follow-up
6. Credentials are device-local — needs TACACS+/RADIUS

> Put this slide in. Examiners ask "what are the weaknesses?" — answering it
> before they ask is worth more than pretending there are none. Know the fix for
> each one; they are in report §9.2 in priority order.

---

## Slide 14 — Conclusion

- All 6 objectives met, verified by test, not by assertion
- The design stages are **not** independent: the addressing decision on slide 4
  produced the routing result on slide 7 **and** the security result on slide 10
- Limitations documented with a prioritised remediation path

**Questions**

---

## Likely questions, and short answers

| Question | Answer |
|---|---|
| Why OSPF and not EIGRP? | EIGRP fits well, but it is Cisco-proprietary and the vendor-independence constraint rules it out. OSPF is RFC 2328. |
| Why areas for only 16 prefixes? | A single area would work. Areas were chosen for the ABR summarisation, and because new branches then scale without touching existing areas. |
| Why /30 and not /31 on p2p links? | /31 is valid (RFC 3021) and saves two addresses per link. /30 was kept for lab clarity; there is no address pressure in a /16. |
| What if HQ-R1 fails? | Both branches and all Internet access go down. Known single point of failure, priced in §9.2: second HQ router, HSRP, dual-homed branches. |
| Why `restrict` and not `shutdown` for port security? | `shutdown` turns every docking-station swap into a physical help-desk visit for 250 users. The violation is still logged. Server ports do use `shutdown`. |
| How does HQ-R1 know which pool to use when DHCP is relayed? | The relay stamps its own SVI address into the **giaddr** field; the router matches giaddr against pool subnets. |
| Is MD5 secure? | Not by modern standards. It raises the bar considerably over nothing. OSPFv3 with IPsec is the correct answer; Packet Tracer cannot demonstrate it. |
| Where would you spend the next dinar? | A stateful firewall at the edge, then IPsec on the WAN. Both are in §9.2 in that order. |
