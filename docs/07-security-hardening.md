# 7. Security Design and Hardening

Every control below is listed with the specific threat it answers. A control
without a named threat is cargo cult, not security.

## 7.1 Control summary

| # | Control | Where | Threat answered |
|---|---|---|---|
| C1 | Edge ingress ACL `ISP-IN` | HQ-R1 Gi0/0 in | Spoofed/inbound Internet traffic, exposure of the static-NAT host |
| C2 | Guest isolation ACL | HQ-CORE Vlan40 in, branch guest subinterfaces in | A compromised or hostile visitor device reaching internal systems |
| C3 | Management ACL + SSH only | all devices, VTY lines | Credential capture over Telnet; device management from user VLANs |
| C4 | Port security (sticky MAC) | all access ports | Unauthorised device on a live port; MAC-flooding of the CAM table |
| C5 | DHCP snooping | access switches | Rogue DHCP server handing out a false gateway (MITM) |
| C6 | BPDU guard + PortFast | all access ports | A user-plugged switch becoming STP root and redirecting traffic |
| C7 | Native VLAN 999, no VLAN 1, trunk pruning, `nonegotiate` | all trunks | VLAN hopping (switch spoofing and double tagging) |
| C8 | Unused ports shut + parked in VLAN 999 | all switches | Opportunistic physical access |
| C9 | OSPF MD5 authentication | all OSPF interfaces | False route injection / routing-based MITM |
| C10 | `service password-encryption`, `enable secret`, local users with `secret` | all devices | Plaintext credentials in a captured configuration file |
| C11 | Login banner + `login block-for` | all devices | Legal notice; brute-force rate limiting |
| C12 | Centralised syslog with timestamps | all devices → 172.16.1.36 | Loss of forensic evidence; inability to correlate events |

## 7.2 C1 — Internet edge ACL (`ISP-IN`)

Applied inbound on `HQ-R1 Gi0/0`:

```
ip access-list extended ISP-IN
 remark --- anti-spoofing: our own and private ranges must never arrive from outside
 deny   ip 172.16.0.0 0.0.255.255 any
 deny   ip 10.0.0.0 0.255.255.255 any
 deny   ip 192.168.0.0 0.0.255.255 any
 deny   ip 127.0.0.0 0.255.255.255 any
 remark --- published services on the static-NAT host only
 permit tcp any host 203.0.113.3 eq 80
 permit tcp any host 203.0.113.3 eq 443
 remark --- return traffic for internal users
 permit tcp any any established
 permit udp any eq 53 any
 permit icmp any any echo-reply
 permit icmp any any time-exceeded
 permit icmp any any unreachable
 deny   ip any any log
```

Reading order matters — an ACL is evaluated top to bottom, first match wins:

1. **Anti-spoofing first.** A packet arriving from the Internet claiming to be
   from `172.16.x.x` is by definition forged, and dropping it first means no
   later `permit` can be tricked into allowing it.
2. **Published services next**, scoped to exactly one host and two ports.
3. **Return traffic** for internally initiated sessions: `established` matches
   TCP segments with ACK/RST set, DNS replies are permitted by source port 53,
   and the three ICMP types keep `ping` and `traceroute` usable for diagnostics
   while blocking inbound `echo-request` (so the edge does not answer scans).
4. **Explicit `deny ip any any log`.** The implicit deny already drops
   everything else; making it explicit adds the `log` keyword, which is what
   turns the ACL into a detection tool as well as a prevention tool.

### Stated limitations

Two rules in `ISP-IN` let return traffic back in without any memory of what went
out. Both are named here rather than left for a reader to find.

**1. `permit tcp any any established`** is a **stateless approximation** of a
stateful firewall. It matches any TCP segment with the ACK bit set, including a
crafted one that belongs to no real session.

**2. `permit udp any eq 53 any` is the wider hole of the two.** UDP has no
handshake and no ACK bit, so there is nothing for the ACL to check: any UDP
datagram whose *source* port is 53 reaches any internal host. An attacker
chooses their own source port, so this is one line of configuration away from an
open inbound UDP path. It is accepted here only because DNS replies have to
return somehow and a stateless ACL has no better tool; narrowing it to the
configured DNS forwarders (`permit udp host <forwarder> eq 53 any`) would reduce
the exposure without a firewall, and is the cheapest available mitigation.

Both are properties of stateless filtering, not of these particular lines. A
production edge needs Cisco Zone-Based Firewall, an ASA/Firepower, or equivalent
stateful inspection, which tracks the outbound flow and permits only the replies
that match it. This is the first item of future work (report §8.2), and is not
claimed as done.

## 7.3 C2 — Guest isolation

Guests must reach the Internet and internal DNS, and nothing else. On
`HQ-CORE`, inbound on the guest SVI:

```
ip access-list extended GUEST-VLAN40-IN
 permit udp any any eq 67
 permit udp any any eq 68
 permit udp any host 172.16.1.34 eq 53
 permit tcp any host 172.16.1.34 eq 53
 deny   ip any 172.16.0.0 0.0.255.255
 permit ip any any
```

The logic is deliberately in this order:

- DHCP first — a client with no address must be able to complete DORA, and
  DISCOVER is sent to `255.255.255.255`, which `deny ip any 172.16.0.0
  0.0.255.255` would not catch but which needs to be permitted before anything
  else can work.
- DNS to **one host, one port** — the narrowest rule that keeps browsing alive.
- Then `deny ip any 172.16.0.0 0.0.255.255` — the whole internal estate,
  including every other site and all infrastructure, in one line. This is the
  payoff from putting everything inside a single /16.
- `permit ip any any` last — anything that is not internal is by definition
  Internet-bound, and is allowed.

Applied inbound on the SVI, so guest traffic is filtered **before** it is
routed — dropped at the first L3 hop, not after crossing the core. The branch
guest VLANs get the same policy on `Gi0/0.140` and `Gi0/0.240`, with the
branch-local DHCP permitted the same way.

Guest-to-guest traffic inside VLAN 40 is *not* blocked by this ACL, because it
never reaches the SVI. If visitor devices must also be isolated from each other,
that requires protected ports (`switchport protected`) or a private-VLAN design
on the access switch. Named, not silently ignored.

## 7.4 C3 — Device management

```
hostname HQ-R1
ip domain-name technova.local
crypto key generate rsa general-keys modulus 1024
username netadmin privilege 15 secret Adm1n#Tech2026
!
ip access-list standard MGMT-ACCESS
 permit 172.16.1.48 0.0.0.15
 deny   any log
!
line vty 0 4
 access-class MGMT-ACCESS in
 transport input ssh
 login local
 exec-timeout 10 0
 logging synchronous
!
ip ssh version 2
login block-for 120 attempts 4 within 60
```

- `transport input ssh` removes Telnet from the VTY lines entirely. Not
  "Telnet discouraged" — Telnet is not listening.
- `access-class MGMT-ACCESS in` means a session is only accepted from the
  management subnet `172.16.1.48/28`. An attacker who compromises a user PC in
  Sales cannot even open a session to a switch.
- `exec-timeout 10 0` closes an idle session after 10 minutes, which addresses
  the unattended-terminal case.
- `login block-for 120 attempts 4 within 60` blocks all login attempts for two
  minutes after four failures in a minute — cheap and effective brute-force
  rate limiting.
- RSA 1024 is the practical maximum in Packet Tracer; a real deployment would
  use 2048 bits minimum.

## 7.5 C4/C5/C6 — Layer-2 controls

### Port security

```
switchport port-security
switchport port-security maximum 2
switchport port-security mac-address sticky
switchport port-security violation restrict
```

Sticky learning writes the first MAC(s) seen into the running configuration, so
the port is bound to the device actually plugged in, with no MAC inventory to
maintain. `maximum 2` accommodates a docking station or a small desk hub;
phone ports use 3. Violation handling is discussed in §4.6.

Secondary benefit: capping MAC addresses per port also defeats **CAM-table
flooding**, the attack where a tool like `macof` fills the MAC table to force
the switch to flood frames and make sniffing possible.

### DHCP snooping

```
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40,60
no ip dhcp snooping information option
interface Port-channel1
 ip dhcp snooping trust
```

Only the uplink toward the legitimate DHCP path is **trusted**. On every other
(untrusted) port, the switch drops DHCP *server* messages — OFFER, ACK — so a
laptop running a rogue DHCP server cannot hand out its own address as the
default gateway. That attack is the easiest full man-in-the-middle available on
a switched LAN, and this is the control that stops it.

`no ip dhcp snooping information option` disables option 82 insertion, which
IOS DHCP relay in this topology would otherwise discard.

### PortFast and BPDU guard

Every access port gets both. PortFast is the performance half (immediate
forwarding, so DHCP does not time out at boot); BPDU guard is the security half
(any BPDU received on that port err-disables it). They belong together: PortFast
alone on a port where someone plugs in a switch is exactly how an unplanned
topology change happens.

## 7.6 C7 — VLAN hopping

Two attacks, two countermeasures:

| Attack | How it works | Countermeasure here |
|---|---|---|
| Switch spoofing | Attacker's host speaks DTP and negotiates a trunk, gaining access to all VLANs | `switchport mode access` on every host port, `switchport mode trunk` + `switchport nonegotiate` on real trunks — DTP is never used |
| Double tagging | Attacker sends a frame with two tags; the first (matching the trunk native VLAN) is stripped, the inner tag delivers the frame into another VLAN | Native VLAN set to unused VLAN 999 on all trunks, and no access port is ever in 999 as an active port — so no attacker's access VLAN matches a native VLAN |

VLAN 1 is used for nothing: not data, not management, not native. It is left
pruned off every trunk.

## 7.7 Password and credential policy

| Item | Value pattern | Notes |
|---|---|---|
| `enable secret` | `NetW0rks#2026` | Type-5/9 hashed, never type-7 |
| Local user | `netadmin` / `Adm1n#Tech2026`, privilege 15 | `secret`, not `password` |
| OSPF MD5 key | `0sPf#Tech26` | Same on both ends of each link |
| `service password-encryption` | enabled | Type-7 obfuscation for anything that must be stored reversibly (e.g. line passwords) — **obfuscation, not encryption**; trivially reversible and no substitute for `secret` |

The credentials above are lab values published in a public portfolio repository. In a
real deployment they would be unique per device, issued from a password manager,
and ideally replaced entirely by TACACS+/RADIUS authentication against a central
identity store — which is listed as future work.

## 7.8 What this design does *not* protect against

Stated plainly, because a security section that claims completeness is wrong:

- **No stateful inspection** at the edge (see §7.2).
- **No IPsec on the WAN.** The serial circuits are leased lines and are treated
  as trusted; anyone with access to the provider's infrastructure sees the
  traffic in clear. A site-to-site IPsec VPN between the routers is the fix.
- **No 802.1X**, so a device that presents an allowed MAC on a live port is
  accepted. Port security is a MAC control, not an identity control.
- **No ARP inspection.** DHCP snooping is configured, which is the prerequisite
  for Dynamic ARP Inspection, but DAI itself is not enabled — so ARP spoofing
  between hosts in the same VLAN remains possible. Enabling
  `ip arp inspection vlan ...` is a two-line follow-up and is in the future-work
  list.
- **No endpoint protection, patching or user-awareness controls.** The network is
  one layer of a security programme, not the whole of it.
