# 3. IP Addressing Plan (VLSM)

## 3.1 Method

The whole internal network is derived from **`172.16.0.0/16`** in two stages:

1. **Stage 1 — site blocks.** The /16 is carved into one contiguous block per
   site, each far larger than the site needs, so that every site can be
   advertised to the rest of the network as a *single summary route*
   (requirement TR-2).
2. **Stage 2 — VLSM inside each site block.** Inside a site block, segments are
   allocated **largest first**, each rounded up to the smallest prefix that
   holds its planned host count. Largest-first ordering is what prevents the
   fragmentation you get from allocating in arbitrary order.

### Stage 1 — site block allocation

| Block | Prefix | Range | Addresses | Purpose |
|---|---|---|---|---|
| HQ | `172.16.0.0/20` | 172.16.0.0 – 172.16.15.255 | 4096 | HQ user, voice, server and management VLANs |
| BR1 | `172.16.16.0/22` | 172.16.16.0 – 172.16.19.255 | 1024 | Irbid branch |
| BR2 | `172.16.20.0/22` | 172.16.20.0 – 172.16.23.255 | 1024 | Aqaba branch |
| Reserved | `172.16.24.0/21` … `172.16.254.0/24` | 172.16.24.0 – 172.16.254.255 | ~59 k | Future sites |
| Infrastructure | `172.16.255.0/24` | 172.16.255.0 – 172.16.255.255 | 256 | WAN /30 links and /32 loopbacks |

Keeping infrastructure links at the very top of the /16 (`.255.0/24`) means
point-to-point links never collide with a future user block, and a single ACL
line (`172.16.255.0 0.0.0.255`) matches all infrastructure.

## 3.2 HQ — VLSM derivation from `172.16.0.0/20`

Allocated largest-first. "Usable" excludes the network and broadcast address.

| # | Segment | Planned hosts | Prefix chosen | Subnet | Usable range | Usable | Spare |
|---|---|---|---|---|---|---|---|
| 1 | Engineering (VLAN 30) | 126 | /25 | `172.16.0.0/25` | 172.16.0.1 – 172.16.0.126 | 126 | 8 |
| 2 | Sales (VLAN 20) | 62 | /26 | `172.16.0.128/26` | 172.16.0.129 – 172.16.0.190 | 62 | 8 |
| 3 | Guest Wi-Fi (VLAN 40) | 62 | /26 | `172.16.0.192/26` | 172.16.0.193 – 172.16.0.254 | 62 | 17 |
| 4 | Voice (VLAN 60) | 62 | /26 | `172.16.2.0/26` | 172.16.2.1 – 172.16.2.62 | 62 | 7 |
| 5 | Administration (VLAN 10) | 30 | /27 | `172.16.1.0/27` | 172.16.1.1 – 172.16.1.30 | 30 | 4 |
| 6 | IT / Servers (VLAN 50) | 14 | /28 | `172.16.1.32/28` | 172.16.1.33 – 172.16.1.46 | 14 | 5 |
| 7 | Device management (VLAN 99) | 14 | /28 | `172.16.1.48/28` | 172.16.1.49 – 172.16.1.62 | 14 | 3 |

**HQ free space after allocation:** `172.16.1.64/26`, `172.16.1.128/25`,
`172.16.2.64/26`, `172.16.2.128/25`, and `172.16.3.0/24` – `172.16.15.255` —
3 712 addresses, comfortably over the 50 % growth requirement (BR-7).

> Note on ordering: Voice is allocated 4th although it is the same size as
> Sales and Guest. It is placed at `172.16.2.0/26` rather than `172.16.1.64/26`
> so that voice sits on its own /24 boundary — this makes future QoS and
> dial-plan ACLs one line (`172.16.2.0 0.0.0.255`) instead of several.

## 3.3 BR1 (Irbid) — VLSM from `172.16.16.0/22`

| # | Segment | Planned hosts | Prefix | Subnet | Usable range | Usable |
|---|---|---|---|---|---|---|
| 1 | BR1 Staff (VLAN 110) | 30 | /27 | `172.16.16.0/27` | 172.16.16.1 – 172.16.16.30 | 30 |
| 2 | BR1 Guest (VLAN 140) | 14 | /28 | `172.16.16.32/28` | 172.16.16.33 – 172.16.16.46 | 14 |
| 3 | BR1 Management (VLAN 199) | 6 | /29 | `172.16.16.48/29` | 172.16.16.49 – 172.16.16.54 | 6 |

Free: `172.16.16.56/29`, `172.16.16.64/26`, `172.16.16.128/25`,
`172.16.17.0/24` – `172.16.19.255`.

## 3.4 BR2 (Aqaba) — VLSM from `172.16.20.0/22`

| # | Segment | Planned hosts | Prefix | Subnet | Usable range | Usable |
|---|---|---|---|---|---|---|
| 1 | BR2 Staff (VLAN 210) | 14 | /28 | `172.16.20.0/28` | 172.16.20.1 – 172.16.20.14 | 14 |
| 2 | BR2 Guest (VLAN 240) | 6 | /29 | `172.16.20.16/29` | 172.16.20.17 – 172.16.20.22 | 6 |
| 3 | BR2 Management (VLAN 299) | 6 | /29 | `172.16.20.24/29` | 172.16.20.25 – 172.16.20.30 | 6 |

Free: `172.16.20.32/27`, `172.16.20.64/26`, `172.16.20.128/25`,
`172.16.21.0/24` – `172.16.23.255`.

## 3.5 Infrastructure — `172.16.255.0/24`

Point-to-point links use /30 (2 usable addresses — exactly what a p2p link
needs; /31 is supported on modern IOS but is avoided here for lab clarity).

| Link | Subnet | Mask | Side A | Side B |
|---|---|---|---|---|
| HQ-R1 ↔ BR1-R1 (serial) | `172.16.255.0/30` | 255.255.255.252 | HQ-R1 Se0/0/0 = **172.16.255.1** | BR1-R1 Se0/0/0 = **172.16.255.2** |
| HQ-R1 ↔ BR2-R1 (serial) | `172.16.255.4/30` | 255.255.255.252 | HQ-R1 Se0/0/1 = **172.16.255.5** | BR2-R1 Se0/0/0 = **172.16.255.6** |
| *reserved for 3rd branch* | `172.16.255.8/30` | — | — | — |
| HQ-R1 ↔ HQ-CORE (routed) | `172.16.255.12/30` | 255.255.255.252 | HQ-CORE Gi0/1 = **172.16.255.13** | HQ-R1 Gi0/1 = **172.16.255.14** |

### Loopbacks (`172.16.255.240/28`)

Loopbacks give each router an always-up management address and a stable OSPF
router-ID source.

| Device | Loopback0 | OSPF router-id |
|---|---|---|
| HQ-R1 | `172.16.255.241/32` | 1.1.1.1 |
| HQ-CORE | `172.16.255.244/32` | 2.2.2.2 |
| BR1-R1 | `172.16.255.242/32` | 3.3.3.3 |
| BR2-R1 | `172.16.255.243/32` | 4.4.4.4 |

Router-IDs are set **statically** rather than letting IOS pick the highest
loopback. A static router-ID never changes when an interface is added or
removed, which keeps `show ip ospf neighbor` output stable across the life of
the network.

## 3.6 Public addressing

| Item | Address | Notes |
|---|---|---|
| ISP-assigned block | `203.0.113.0/29` | 6 usable: .1 – .6 |
| ISP-R Gi0/0 | `203.0.113.1/29` | Provider side; HQ default gateway |
| HQ-R1 Gi0/0 | `203.0.113.2/29` | PAT address for all internal users |
| Web/app server (static NAT) | `203.0.113.3` | Translates to `172.16.1.35` |
| Spare public | `203.0.113.4 – .6` | Held for a future mail/VPN endpoint |
| Simulated Internet segment | `198.51.100.0/24` | ISP-R Gi0/1 = .1, external test server = .10 |

`203.0.113.0/24` and `198.51.100.0/24` are RFC 5737 documentation ranges — the
correct thing to use in a written design, instead of squatting on address space
that belongs to a real organisation.

## 3.7 Master address table — gateways and static hosts

| Site | VLAN | Name | Subnet | Mask | Gateway (SVI / subif) | Static hosts |
|---|---|---|---|---|---|---|
| HQ | 10 | ADMIN | 172.16.1.0/27 | 255.255.255.224 | 172.16.1.1 | — (DHCP) |
| HQ | 20 | SALES | 172.16.0.128/26 | 255.255.255.192 | 172.16.0.129 | — (DHCP) |
| HQ | 30 | ENGINEERING | 172.16.0.0/25 | 255.255.255.128 | 172.16.0.1 | — (DHCP) |
| HQ | 40 | GUEST | 172.16.0.192/26 | 255.255.255.192 | 172.16.0.193 | — (DHCP) |
| HQ | 50 | IT-SERVERS | 172.16.1.32/28 | 255.255.255.240 | 172.16.1.33 | `.34` DNS, `.35` Web/App, `.36` Syslog/NTP/TFTP |
| HQ | 60 | VOICE | 172.16.2.0/26 | 255.255.255.192 | 172.16.2.1 | — (DHCP + option 150) |
| HQ | 99 | MGMT | 172.16.1.48/28 | 255.255.255.240 | 172.16.1.49 (HQ-CORE SVI) | `.50` HQ-ACC1, `.51` HQ-ACC2 |
| HQ | 999 | PARKING | — | — | none (no SVI) | unused ports only |
| BR1 | 110 | BR1-STAFF | 172.16.16.0/27 | 255.255.255.224 | 172.16.16.1 | — (DHCP) |
| BR1 | 140 | BR1-GUEST | 172.16.16.32/28 | 255.255.255.240 | 172.16.16.33 | — (DHCP) |
| BR1 | 199 | BR1-MGMT | 172.16.16.48/29 | 255.255.255.248 | 172.16.16.49 | `.50` BR1-SW1 |
| BR2 | 210 | BR2-STAFF | 172.16.20.0/28 | 255.255.255.240 | 172.16.20.1 | — (DHCP) |
| BR2 | 240 | BR2-GUEST | 172.16.20.16/29 | 255.255.255.248 | 172.16.20.17 | — (DHCP) |
| BR2 | 299 | BR2-MGMT | 172.16.20.24/29 | 255.255.255.248 | 172.16.20.25 | `.26` BR2-SW1 |

**Convention used throughout:** the gateway is always the **first usable
address** of the subnet, servers take the lowest addresses after the gateway,
and DHCP pools start at least 20 addresses above the gateway so static
assignments can grow without a pool conflict.

## 3.8 Wildcard masks (for OSPF and ACL statements)

| Prefix | Subnet mask | Wildcard |
|---|---|---|
| /22 | 255.255.252.0 | 0.0.3.255 |
| /20 | 255.255.240.0 | 0.0.15.255 |
| /25 | 255.255.255.128 | 0.0.0.127 |
| /26 | 255.255.255.192 | 0.0.0.63 |
| /27 | 255.255.255.224 | 0.0.0.31 |
| /28 | 255.255.255.240 | 0.0.0.15 |
| /29 | 255.255.255.248 | 0.0.0.7 |
| /30 | 255.255.255.252 | 0.0.0.3 |
| /32 | 255.255.255.255 | 0.0.0.0 |
