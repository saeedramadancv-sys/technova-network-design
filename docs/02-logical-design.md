# 2. Logical Design

## 2.1 Design model

The network follows the classic **three-layer hierarchical model**, collapsed
where the site size does not justify a separate layer:

| Site | Core | Distribution | Access |
|---|---|---|---|
| HQ | `HQ-R1` (border + WAN aggregation) | `HQ-CORE` — multilayer switch, collapsed core/distribution | `HQ-ACC1`, `HQ-ACC2` |
| BR1 | — | `BR1-R1` (router-on-a-stick) | `BR1-SW1` |
| BR2 | — | `BR2-R1` (router-on-a-stick) | `BR2-SW1` |

**Why collapsed at HQ:** a dedicated distribution layer is justified when the
access-switch count exceeds roughly 8–10 per block. HQ has two access switches,
so a single multilayer switch performing both roles removes two devices, two
hops of latency, and a whole spanning-tree domain, without creating a capacity
problem.

**Why router-on-a-stick at the branches:** each branch has one access switch and
at most 50 users. Inter-VLAN traffic there is a few Mbps, far below what a
router subinterface handles, and a multilayer switch per branch would roughly
double branch hardware cost for no measurable benefit. This is documented as a
deliberate cost trade-off, not an oversight — see §5.4 of the final report for
the throughput comparison.

## 2.2 WAN topology

Hub-and-spoke, with HQ as the hub:

```
                       ┌──────────┐
                       │  ISP-R   │  203.0.113.0/29
                       └────┬─────┘  (simulated Internet)
                            │ Gi0/0
                       ┌────┴─────┐
             Se0/0/0   │  HQ-R1   │   Se0/0/1
        ┌──────────────┤  (ABR)   ├──────────────┐
        │              └────┬─────┘              │
        │ 172.16.255.0/30   │ Gi0/1              │ 172.16.255.4/30
        │                   │ 172.16.255.12/30   │
   ┌────┴─────┐        ┌────┴──────┐        ┌────┴─────┐
   │ BR1-R1   │        │  HQ-CORE  │        │ BR2-R1   │
   │ Area 1   │        │  Area 0   │        │ Area 2   │
   └────┬─────┘        └──┬─────┬──┘        └────┬─────┘
        │              Po1│     │Po2             │
   ┌────┴─────┐     ┌─────┴─┐ ┌─┴─────┐     ┌────┴─────┐
   │ BR1-SW1  │     │HQ-ACC1│ │HQ-ACC2│     │ BR2-SW1  │
   └──────────┘     └───────┘ └───────┘     └──────────┘
```

A full mesh (adding a direct BR1↔BR2 circuit) was considered and rejected: it
adds a third leased line to carry branch-to-branch traffic that is measured in
kilobits, and it would place a link between two non-backbone areas, requiring an
OSPF virtual link. Hub-and-spoke keeps the area design clean. The honest cost is
that HQ is a single point of failure for inter-site traffic; §8.2 of the report
prices the dual-hub alternative.

## 2.3 Device roles

| Device | Model (Packet Tracer) | Role |
|---|---|---|
| `ISP-R` | 2911 | Simulates the service provider; default gateway of last resort, hosts the external test server |
| `HQ-R1` | 2911 + HWIC-2T | Internet border, NAT/PAT, edge ACL, WAN aggregation, OSPF **ABR** for areas 1 and 2, central DHCP server for HQ |
| `HQ-CORE` | 3560-24PS | Inter-VLAN routing for all HQ VLANs (SVIs), STP root bridge, EtherChannel aggregation, DHCP relay, guest ACL enforcement |
| `HQ-ACC1` | 2960-24TT | Access for Engineering / Sales / Administration + voice |
| `HQ-ACC2` | 2960-24TT | Access for Guest Wi-Fi and the IT/server segment |
| `BR1-R1` | 2911 + HWIC-2T | BR1 inter-VLAN routing (802.1Q subinterfaces), local DHCP, OSPF area 1 |
| `BR1-SW1` | 2960-24TT | BR1 access layer |
| `BR2-R1` | 2911 + HWIC-2T | BR2 inter-VLAN routing, local DHCP, OSPF area 2 |
| `BR2-SW1` | 2960-24TT | BR2 access layer |

## 2.4 Traffic flows and enforcement points

| Flow | Path | Where policy is enforced |
|---|---|---|
| HQ user → HQ server | ACC → CORE (SVI routing) → ACC | `HQ-CORE` SVI ACLs |
| HQ guest → Internet | ACC2 → CORE (VLAN 40 SVI, ACL) → HQ-R1 (PAT) → ISP | `GUEST-VLAN40-IN` on `HQ-CORE`, then PAT on `HQ-R1` |
| Branch user → HQ server | SW → branch router subif → serial → HQ-R1 → CORE → server | branch subinterface ACL + `HQ-CORE` |
| Branch user → Internet | branch router → serial → HQ-R1 (PAT) → ISP | PAT on `HQ-R1` (all Internet traffic is centrally inspected) |
| Internet → web server | ISP → HQ-R1 (`ISP-IN` ACL + static NAT) → CORE → VLAN 50 | `ISP-IN` ACL, static NAT entry |
| Branch user → other branch | SW → router → serial → HQ-R1 → serial → router → SW | routed only, no NAT (internal) |

Centralising Internet egress at HQ is a deliberate security decision: there is
exactly **one** place where the company touches the Internet, so there is
exactly one place to filter, log and later upgrade to a real firewall.

## 2.5 Failure behaviour

| Failure | Effect | Recovery |
|---|---|---|
| One core↔access link | None visible; the LACP bundle keeps forwarding on the survivor | Sub-second, hardware |
| `HQ-ACC1` | Its attached users lose access | Manual; single-homed access switches are accepted (users are single-homed anyway) |
| Branch WAN circuit | Branch loses HQ and Internet; **stays up internally** because DHCP and inter-VLAN routing are local (TR-7) | Circuit repair; designed backup in report §8.2 |
| `HQ-R1` | All inter-site and Internet traffic stops; HQ internal traffic unaffected | Single point of failure — accepted in scope, priced in report §8.2 |
| ISP link | Internet down for all sites; inter-site traffic unaffected | Second ISP + BGP or floating static — future work |
