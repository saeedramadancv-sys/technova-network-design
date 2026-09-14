# 5. Routing Design

## 5.1 Protocol selection

| Candidate | Verdict | Reason |
|---|---|---|
| Static routing | Rejected | ~16 internal prefixes across 4 routers; every future subnet becomes a manual change on every router, and there is no automatic failover |
| RIPv2 | Rejected | 15-hop limit is irrelevant here, but 30-second periodic updates and counting-to-infinity convergence are poor, and it cannot express areas |
| EIGRP | Rejected | Technically an excellent fit (fast, simple, supports unequal-cost load balancing), but it is Cisco-proprietary and constraint §1.5 requires an open standard |
| **OSPFv2** | **Selected** | Open standard (RFC 2328), link-state with fast convergence, hierarchical areas, supports authentication and route summarisation at area borders |

Process ID `10` is used on every device. The process ID is only locally
significant, but keeping it identical everywhere removes a pointless source of
confusion during troubleshooting.

## 5.2 Area design

```
        ┌─────────────── Area 0 (backbone) ───────────────┐
        │   HQ-CORE (2.2.2.2) ──── HQ-R1 (1.1.1.1)        │
        │   all HQ VLANs          Lo0, ISP edge           │
        └───────────┬──────────────────────┬──────────────┘
              Area 1│                       │Area 2
          ┌─────────┴────────┐    ┌─────────┴────────┐
          │ BR1-R1 (3.3.3.3) │    │ BR2-R1 (4.4.4.4) │
          │ VLANs 110/140/199│    │ VLANs 210/240/299│
          └──────────────────┘    └──────────────────┘
```

| Area | Members | Contents |
|---|---|---|
| 0 (backbone) | HQ-R1, HQ-CORE | All HQ VLAN subnets, the CORE↔R1 link, HQ-R1 and HQ-CORE loopbacks |
| 1 | HQ-R1 (ABR), BR1-R1 | BR1 serial link, BR1 VLANs, BR1-R1 loopback |
| 2 | HQ-R1 (ABR), BR2-R1 | BR2 serial link, BR2 VLANs, BR2-R1 loopback |

`HQ-R1` is therefore an **ABR** (Area Border Router) for both branch areas and
simultaneously an **ASBR** (Autonomous System Boundary Router), because it
injects the default route learned from the ISP.

**Why areas at all, at this size?** Honestly, a single area 0 would work fine for
16 prefixes — and that is worth saying plainly rather than pretending areas are
mandatory here. Areas are used because:

1. They make the design scale in the direction the company is actually growing
   (new branches = new areas, each added without touching the others).
2. They allow **summarisation at the ABR**, which is the concrete, measurable
   benefit shown in §5.3.
3. A branch link flap only regenerates LSAs inside that branch area, instead of
   causing an SPF run on every router in the company.

## 5.3 Route summarisation

The addressing plan was built so that each branch occupies exactly one /22.
That makes the ABR configuration two lines:

```
router ospf 10
 area 1 range 172.16.16.0 255.255.252.0
 area 2 range 172.16.20.0 255.255.252.0
```

Effect, measured on `HQ-CORE`:

| | Without `area range` | With `area range` |
|---|---|---|
| Inter-area routes for BR1 | 4 (`/27`, `/28`, `/29`, `/30`) | **1** (`172.16.16.0/22`) |
| Inter-area routes for BR2 | 4 | **1** |
| Total branch LSA-3 entries in area 0 | 8 | **2** |

A 75 % reduction in the branch routes carried into the backbone, and — more
importantly — the backbone stops hearing about an individual branch VLAN going
away. Only the loss of the *entire* /22 is propagated.

Verification: `show ip route ospf` on HQ-CORE should show `172.16.16.0/22 [110/...]`
and `172.16.20.0/22 [110/...]` as `O IA` routes, and no more specific branch
prefixes.

## 5.4 Cost and path selection

OSPF cost is `reference-bandwidth / interface-bandwidth`, with the IOS default
reference bandwidth of 100 Mbps.

| Link | Bandwidth | Default cost |
|---|---|---|
| Gi0/1 CORE↔R1 | 1 Gbps | 1 (floor) |
| Po1 / Po2 | 200 Mbps | 1 (floor) |
| Serial to branches | 2 Mbps (`bandwidth 2000`) | 50 |
| SVIs / LAN | 100 Mbps – 1 Gbps | 1 |

Because every link of 100 Mbps and above computes to the minimum cost of 1, the
default reference bandwidth **cannot distinguish a 100 Mbps link from a 1 Gbps
link**. `auto-cost reference-bandwidth 10000` (10 Gbps) is therefore configured
on all four OSPF routers, which yields:

| Link | Cost with reference 10 Gbps |
|---|---|
| 1 Gbps routed link | 10 |
| 200 Mbps EtherChannel | 50 |
| 100 Mbps access link | 100 |
| 2 Mbps serial | 5000 |

The reference bandwidth **must** be identical on every router in the domain —
mismatched values produce asymmetric routing that is painful to diagnose. This
is why it appears in all four configurations and in the test plan (test 5.4).

There is no equal-cost path to design for: the topology is a tree. The single
place where load sharing happens is the LACP bundle, which is a Layer-2
mechanism and invisible to OSPF.

## 5.5 Default route and Internet reachability

On `HQ-R1` only:

```
ip route 0.0.0.0 0.0.0.0 203.0.113.1
router ospf 10
 default-information originate
```

`default-information originate` advertises the default as an **external type-2
(E2)** LSA into OSPF, so every router at every site learns
`O*E2 0.0.0.0/0 [110/1] via ...` and no branch needs a static route. E2 (rather
than E1) means the metric is not incremented per hop, which is correct when
there is only one exit point — there is nothing to compare.

`passive-interface GigabitEthernet0/0` is set so that OSPF hellos are never sent
to the ISP. Sending routing protocol packets to a provider is both noise and an
information leak.

## 5.6 Authentication

OSPF adjacencies are authenticated with MD5 on every non-passive interface:

```
interface Serial0/0/0
 ip ospf message-digest-key 1 md5 0sPf#Tech26
router ospf 10
 area 0 authentication message-digest
 area 1 authentication message-digest
 area 2 authentication message-digest
```

This stops an attacker with physical access to a link from forming an adjacency
and injecting a false default route — the cheapest possible full
man-in-the-middle. MD5 is weak as a hash by modern standards; the accurate
statement is that this raises the bar considerably over no authentication, and
that a current design would use OSPFv3 with IPsec AH/ESP. Packet Tracer does not
implement SHA-family OSPF authentication, so MD5 is what the build can
demonstrate.

## 5.7 Expected routing table (HQ-CORE, steady state)

| Prefix | Type | Next hop | Notes |
|---|---|---|---|
| 172.16.0.0/25, 172.16.0.128/26, 172.16.0.192/26, 172.16.1.0/27, 172.16.1.32/28, 172.16.1.48/28, 172.16.2.0/26 | C / L | directly connected SVIs | HQ VLANs |
| 172.16.255.12/30 | C | Gi0/1 | link to HQ-R1 |
| 172.16.16.0/22 | O IA | 172.16.255.14 | **summarised** BR1 |
| 172.16.20.0/22 | O IA | 172.16.255.14 | **summarised** BR2 |
| 172.16.255.0/30, 172.16.255.4/30 | O IA | 172.16.255.14 | WAN links (inside the /22 summary? no — these are in the .255 block, advertised individually) |
| 172.16.255.241/32, .242/32, .243/32 | O / O IA | 172.16.255.14 | loopbacks |
| 0.0.0.0/0 | O*E2 | 172.16.255.14 | default from HQ-R1 |

> Note on the WAN /30s: they live in `172.16.255.0/24`, not inside a branch /22,
> so the `area range` statements do not cover them and they appear as individual
> inter-area routes. That is intentional — keeping p2p links in a separate block
> means a link address is never hidden behind a site summary, which makes
> hop-by-hop traceroute output readable. If suppressing them is ever wanted, a
> second `area X range ... not-advertise` would do it.
