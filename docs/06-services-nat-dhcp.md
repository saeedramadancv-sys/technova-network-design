# 6. Network Services: DHCP, DNS, NAT, NTP, Syslog

## 6.1 DHCP architecture

A single design decision drives everything here: **branches must survive a WAN
outage** (requirement BR-5 / TR-7).

| Site | DHCP server | Why |
|---|---|---|
| HQ | `HQ-R1` (IOS DHCP), relayed by `HQ-CORE` SVIs | Central management; the DHCP server and the clients are in the same site, so a WAN failure is irrelevant |
| BR1 | `BR1-R1`, local pools | If the serial link drops, hosts still get addresses and keep working locally |
| BR2 | `BR2-R1`, local pools | Same |

Centralising branch DHCP at HQ would have been less configuration, and is what
many designs do — but it makes a WAN outage escalate from "no HQ access" to "no
network at all" as soon as the first lease expires. That trade is explicitly
rejected here.

### DHCP relay at HQ

Clients broadcast; the HQ pools live on `HQ-R1`, one hop away. Each HQ SVI on
`HQ-CORE` carries:

```
interface Vlan30
 ip address 172.16.0.1 255.255.255.128
 ip helper-address 172.16.255.14
```

`ip helper-address` converts the client's broadcast DISCOVER into a unicast to
`HQ-R1`. `HQ-R1` reads the **giaddr** field that the relay stamped in — that is
how it knows which pool to answer from, without any VLAN awareness of its own.

### Pools

| Pool name | Network | Excluded | Gateway | DNS | Extra options |
|---|---|---|---|---|---|
| `HQ-ENG-V30` | 172.16.0.0/25 | .1 – .20 | 172.16.0.1 | 172.16.1.34 | domain `technova.local` |
| `HQ-SALES-V20` | 172.16.0.128/26 | .129 – .140 | 172.16.0.129 | 172.16.1.34 | — |
| `HQ-GUEST-V40` | 172.16.0.192/26 | .193 – .200 | 172.16.0.193 | 172.16.1.34 | lease 0 4 0 (4 hours) |
| `HQ-ADMIN-V10` | 172.16.1.0/27 | .1 – .10 | 172.16.1.1 | 172.16.1.34 | — |
| `HQ-VOICE-V60` | 172.16.2.0/26 | .1 – .10 | 172.16.2.1 | 172.16.1.34 | option 150 → 172.16.1.36 (TFTP) |
| `BR1-STAFF-V110` | 172.16.16.0/27 | .1 – .10 | 172.16.16.1 | 172.16.1.34 | on BR1-R1 |
| `BR1-GUEST-V140` | 172.16.16.32/28 | .33 – .36 | 172.16.16.33 | 172.16.1.34 | on BR1-R1, lease 4 h |
| `BR2-STAFF-V210` | 172.16.20.0/28 | .1 – .4 | 172.16.20.1 | 172.16.1.34 | on BR2-R1 |
| `BR2-GUEST-V240` | 172.16.20.16/29 | .17 – .18 | 172.16.20.17 | 172.16.1.34 | on BR2-R1, lease 4 h |

Design details worth defending in a viva:

- **Exclusions before the pool is used, not after.** `ip dhcp excluded-address`
  must be configured *before* the pool statement in IOS ordering terms, and the
  excluded range always covers the gateway plus room for future statics.
- **Option 150 for voice.** IP phones fetch their firmware and dial plan from a
  TFTP server given in DHCP option 150. Without it, phones register but stay
  unconfigured.
- **Short guest leases (4 h).** Guests are transient. A 24-hour default lease on
  a 62-address guest pool exhausts it during a busy conference day; 4 hours
  recycles addresses at roughly the rate visitors actually leave.
- **VLAN 50 (servers) and VLAN 99 (management) have no pool.** Infrastructure and
  servers are statically addressed, on purpose: a server whose address can change
  breaks every ACL and DNS record that points at it.

## 6.2 DNS

- Internal authoritative server: `172.16.1.34`, zone `technova.local`.
- Records used by the test plan: `www.technova.local → 172.16.1.35`,
  `syslog.technova.local → 172.16.1.36`.
- The external test server (`198.51.100.10`) is resolved by the ISP-side DNS in
  the simulation; in Packet Tracer it is simplest to reach it by address and use
  the internal server for name tests.
- Guests are permitted to reach `172.16.1.34` for DNS **and nothing else
  internal** (see §7.3). Blocking guest DNS entirely would break their Internet
  access; permitting all of VLAN 50 would expose the servers. One host, one
  protocol, one port is the correct width for that rule.

## 6.3 NAT and PAT

All internal sites reach the Internet through `HQ-R1`.

### Interface roles

| Interface | NAT role |
|---|---|
| Gi0/0 (to ISP) | `ip nat outside` |
| Gi0/1 (to HQ-CORE) | `ip nat inside` |
| Se0/0/0 (to BR1) | `ip nat inside` |
| Se0/0/1 (to BR2) | `ip nat inside` |

Marking *both* serial interfaces as inside is what lets branch traffic be
translated; forgetting one of them is the single most common reason "the branch
can ping HQ but not the Internet".

### Dynamic PAT (many-to-one)

```
ip access-list standard NAT-ALLOWED
 permit 172.16.0.0 0.0.15.255     ! HQ   /20
 permit 172.16.16.0 0.0.3.255     ! BR1  /22
 permit 172.16.20.0 0.0.3.255     ! BR2  /22
!
ip nat inside source list NAT-ALLOWED interface GigabitEthernet0/0 overload
```

`overload` is PAT: every internal host shares `203.0.113.2`, distinguished by
source port. With one public address and 320 users, this is not a preference —
it is the only option (requirement BR-2).

The ACL is written as **three site summaries rather than one `172.16.0.0
0.0.255.255`**. The difference matters: the narrow version does not translate
`172.16.255.0/24`, so infrastructure and loopback addresses can never
accidentally source Internet traffic, and adding a new site is a deliberate
one-line change rather than something that happens automatically.

### Static NAT (one-to-one, inbound)

```
ip nat inside source static 172.16.1.35 203.0.113.3
```

This is bidirectional: outside hosts reach the web server at `203.0.113.3`, and
the server's own outbound traffic uses `203.0.113.3` rather than the PAT address.
It must be paired with the `ISP-IN` ACL (§7.2) permitting only TCP 80/443 to that
address — a static NAT without an inbound filter publishes the whole host.

### Verification

| Command | What it proves |
|---|---|
| `show ip nat translations` | Live translation table; look for `icmp 203.0.113.2:x` entries with different inside locals |
| `show ip nat statistics` | Hit/miss counters and which ACL and interfaces are bound |
| `debug ip nat` | Per-packet translation, for a failing flow only |

## 6.4 Time (NTP)

```
! HQ-R1 — authoritative for the network
ntp master 3
! every other device
ntp server 172.16.255.241
clock timezone AST 3
service timestamps log datetime msec localtime
```

Synchronised clocks are not cosmetic. Without them, correlating a syslog entry on
an access switch with one on the border router is guesswork, and any future
certificate or Kerberos-based service fails outright. `HQ-R1` is the master
because it is the device that will later take a real upstream NTP source from the
ISP.

## 6.5 Logging (Syslog)

```
logging host 172.16.1.36
logging trap informational
logging source-interface Loopback0      ! routers
service timestamps log datetime msec localtime
```

- Severity `informational` (6) captures interface state changes, port-security
  violations, ACL `log` hits and configuration changes, without the volume of
  `debugging` (7).
- `logging source-interface Loopback0` makes every message arrive from a stable
  address, so the log server identifies the device by one IP regardless of which
  path the message took.
- Local buffer as well: `logging buffered 16384` so the last messages survive a
  syslog-server outage and can be read with `show logging`.

## 6.6 Service summary table

| Service | Where it runs | Address | Clients |
|---|---|---|---|
| DHCP (HQ) | HQ-R1 | 172.16.255.14 (relay target) | HQ VLANs 10/20/30/40/60 |
| DHCP (BR1) | BR1-R1 | local | VLANs 110, 140 |
| DHCP (BR2) | BR2-R1 | local | VLANs 210, 240 |
| DNS | server in VLAN 50 | 172.16.1.34 | all internal hosts + guests |
| Web / app | server in VLAN 50 | 172.16.1.35 → 203.0.113.3 | internal + Internet |
| Syslog / NTP / TFTP | server in VLAN 50 | 172.16.1.36 | all network devices |
| NAT / PAT | HQ-R1 | 203.0.113.2 | all sites |
