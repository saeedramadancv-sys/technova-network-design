# 4. VLAN and Layer-2 Switching Design

## 4.1 VLAN numbering scheme

VLAN IDs are not arbitrary. The scheme encodes the site and the function so that
any engineer reading a trunk configuration knows immediately where the traffic
belongs:

| Range | Meaning |
|---|---|
| 10 – 99 | HQ VLANs (function-numbered: 10 admin, 20 sales, 30 engineering, 40 guest, 50 servers, 60 voice, 99 management) |
| 110 – 199 | BR1 VLANs — HQ function number + 100 |
| 210 – 299 | BR2 VLANs — HQ function number + 200 |
| 999 | PARKING — the black-hole VLAN for unused access ports |

Because branch VLANs are locally significant (no VLAN spans the WAN), reusing
`10/20/40/99` at every site would also have worked. Offsetting by site removes an
entire class of human error: a misconfigured trunk or a mistakenly bridged link
cannot silently merge two sites' user VLANs, because their IDs do not match.

## 4.2 VLAN table

### HQ

| VLAN | Name | Subnet | Present on |
|---|---|---|---|
| 10 | ADMIN | 172.16.1.0/27 | HQ-CORE, HQ-ACC1 |
| 20 | SALES | 172.16.0.128/26 | HQ-CORE, HQ-ACC1 |
| 30 | ENGINEERING | 172.16.0.0/25 | HQ-CORE, HQ-ACC1 |
| 40 | GUEST | 172.16.0.192/26 | HQ-CORE, HQ-ACC2 |
| 50 | IT-SERVERS | 172.16.1.32/28 | HQ-CORE, HQ-ACC2 |
| 60 | VOICE | 172.16.2.0/26 | HQ-CORE, HQ-ACC1 |
| 99 | MGMT | 172.16.1.48/28 | HQ-CORE, HQ-ACC1, HQ-ACC2 |
| 999 | PARKING | none | HQ-ACC1, HQ-ACC2 |

### Branches

| VLAN | Name | Subnet | Present on |
|---|---|---|---|
| 110 | BR1-STAFF | 172.16.16.0/27 | BR1-R1 (Gi0/0.110), BR1-SW1 |
| 140 | BR1-GUEST | 172.16.16.32/28 | BR1-R1 (Gi0/0.140), BR1-SW1 |
| 199 | BR1-MGMT | 172.16.16.48/29 | BR1-R1 (Gi0/0.199), BR1-SW1 |
| 210 | BR2-STAFF | 172.16.20.0/28 | BR2-R1 (Gi0/0.210), BR2-SW1 |
| 240 | BR2-GUEST | 172.16.20.16/29 | BR2-R1 (Gi0/0.240), BR2-SW1 |
| 299 | BR2-MGMT | 172.16.20.24/29 | BR2-R1 (Gi0/0.299), BR2-SW1 |

## 4.3 Trunking

All switch-to-switch and switch-to-router links are 802.1Q trunks with an
explicitly pruned allowed-VLAN list.

| Trunk | Endpoints | Allowed VLANs | Native VLAN |
|---|---|---|---|
| Po1 | HQ-CORE Fa0/1-2 ↔ HQ-ACC1 Fa0/1-2 | 10,20,30,60,99 | 999 |
| Po2 | HQ-CORE Fa0/3-4 ↔ HQ-ACC2 Fa0/1-2 | 40,50,99 | 999 |
| ROAS | BR1-R1 Gi0/0 ↔ BR1-SW1 Fa0/1 | 110,140,199 | 999 |
| ROAS | BR2-R1 Gi0/0 ↔ BR2-SW1 Fa0/1 | 210,240,299 | 999 |

Three deliberate choices here:

1. **`switchport mode trunk` with DTP negotiation disabled**
   (`switchport nonegotiate`). Trunks are static on both ends. Dynamic
   negotiation is what makes VLAN-hopping via a rogue DTP-speaking device
   possible.
2. **Explicit allowed-VLAN lists.** A trunk that carries only what it must is
   both a security control and a broadcast-domain optimisation: guest frames
   never appear on the Engineering access switch at all.
3. **Native VLAN moved to 999.** VLAN 1 is never used for data, management or
   as a native VLAN. VLAN 999 carries no subnet and no SVI, so untagged frames
   that arrive on a trunk land nowhere. This closes the classic
   double-tagging attack, which depends on the attacker's access VLAN matching
   the trunk native VLAN.

## 4.4 EtherChannel (HQ only)

HQ-CORE reaches each access switch over **two** links bundled with LACP:

```
HQ-CORE Fa0/1 ──┐                  ┌── HQ-ACC1 Fa0/1
                ├── Po1 (LACP) ────┤
HQ-CORE Fa0/2 ──┘                  └── HQ-ACC1 Fa0/2
```

- Mode: `channel-group N mode active` on both ends — LACP (802.3ad), the open
  standard, in preference to Cisco's proprietary PAgP.
- Load balancing: `src-dst-ip` on HQ-CORE, so flows between the same pair of
  hosts are spread across members by IP pair rather than by MAC (access-layer
  MACs are dominated by the gateway MAC, which would pin almost all traffic to
  one member).
- Spanning tree sees **one** logical port, so both physical links forward. That
  is the point: with plain redundant links, STP would block one of them and half
  the purchased capacity would sit idle until a failure.

Both member ports must be configured identically (speed, duplex, mode, allowed
VLANs) or LACP refuses to bundle them — a frequent build error, so the build
guide configures the range, not individual ports.

## 4.5 Spanning tree

| Setting | Value | Reason |
|---|---|---|
| Mode | `rapid-pvst` (802.1w per VLAN) | Converges in a few seconds instead of ~50 s for classic 802.1D |
| Root bridge | `HQ-CORE`, `root primary` for all HQ VLANs | The root must be the device that is topologically central and in the traffic path; letting the lowest-MAC access switch win by accident produces a badly shaped tree |
| Access ports | `spanning-tree portfast` | A host port has no loop risk; PortFast skips listening/learning so DHCP works on boot |
| Access ports | `spanning-tree bpduguard enable` | If any BPDU ever arrives on a host port, the port goes err-disabled. This is the control that stops a user plugging in an unmanaged switch and becoming root |
| Trunks | default (no PortFast) | Trunks are real potential loops and must run the full state machine |

## 4.6 Access-port template

Every access port follows one of three templates. This is what makes the
configurations short and auditable.

**Template A — data host (e.g. HQ-ACC1 Fa0/5):**
```
 switchport mode access
 switchport access vlan 30
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
```

**Template B — data host + IP phone (e.g. HQ-ACC1 Fa0/15):**
```
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 60
 switchport port-security
 switchport port-security maximum 3
 ... (as above)
```
`maximum 3` allows the phone, the PC behind it, and one replacement without a
violation; the phone tags voice frames with VLAN 60 while PC frames stay
untagged in VLAN 10.

**Template C — unused port:**
```
 switchport mode access
 switchport access vlan 999
 shutdown
```
Administratively down *and* parked in a VLAN with no gateway — two independent
reasons a patched-in cable achieves nothing.

### On `violation restrict` vs `shutdown`

`restrict` drops offending frames and raises a syslog message; `shutdown`
err-disables the whole port. `restrict` is chosen because in a 250-user office,
`shutdown` converts every docking-station or hub swap into a help-desk ticket and
a physical visit. The security event is still logged, which is what the control
is actually for. A server or comms-room port would justify `shutdown`.
