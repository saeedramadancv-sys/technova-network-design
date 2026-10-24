# 8. Test Plan and Verification

Every test states **where** it is run, **what** is run, and the **exact expected
result**. A test whose expected result is "it should work" is not a test.

Record the actual result in the right-hand column when you run the build; that
filled-in table is the evidence section of the report.

---

## Group A — Layer 1 / Layer 2

| # | Device | Command | Expected result | Actual |
|---|---|---|---|---|
| A1 | all switches | `show ip interface brief` | every configured interface `up/up`; unused ports `administratively down` | |
| A2 | HQ-CORE | `show vlan brief` | VLANs 10, 20, 30, 40, 50, 60, 99, 999 present and named | |
| A3 | HQ-CORE | `show interfaces trunk` | Po1 and Po2 listed; native VLAN **999**; allowed lists are 10,20,30,60,99 and 40,50,99 — **not** 1-4094 | |
| A4 | HQ-CORE | `show etherchannel summary` | `Po1(SU)` and `Po2(SU)`; each member port flagged `(P)` = bundled | |
| A5 | HQ-ACC1 | `show spanning-tree vlan 30` | root bridge ID = HQ-CORE; `Po1` is the **root port**; no blocked ports | |
| A6 | HQ-CORE | `show spanning-tree summary` | this switch is root for all HQ VLANs; mode `rapid-pvst` | |
| A7 | HQ-ACC1 | `show port-security` | each active access port: max 2 (or 3 on voice ports), `SecurityViolation 0`, action `Restrict` | |
| A8 | HQ-ACC1 | `show port-security address` | sticky MAC entries appear after hosts have sent traffic | |

**A4 troubleshooting:** if a member shows `(I)` = individual, the two ends have
different settings. Compare `show run interface Fa0/1` and `Fa0/2` on both
switches — mode, native VLAN, allowed list and speed must match exactly.

---

## Group B — Addressing and DHCP

| # | Device | Command | Expected result | Actual |
|---|---|---|---|---|
| B1 | PC in VLAN 30 | `ipconfig /all` (PC Desktop → Command Prompt) | address in 172.16.0.21–126, mask 255.255.255.128, gateway 172.16.0.1, DNS 172.16.1.34 | |
| B2 | PC in VLAN 20 | `ipconfig` | address in 172.16.0.141–190, gateway 172.16.0.129 | |
| B3 | PC in VLAN 40 (guest) | `ipconfig` | address in 172.16.0.201–254, gateway 172.16.0.193 | |
| B4 | HQ-R1 | `show ip dhcp binding` | one binding per DHCP client, with the correct pool subnet | |
| B5 | HQ-R1 | `show ip dhcp pool` | per pool: leased count > 0, no `excluded` conflicts | |
| B6 | BR1 PC | `ipconfig` | 172.16.16.11–30, gateway 172.16.16.1 — **served by BR1-R1, not HQ** | |
| B7 | BR2 PC | `ipconfig` | 172.16.20.5–14, gateway 172.16.20.1 | |
| B8 | HQ-CORE | `show run interface Vlan30` | contains `ip helper-address 172.16.255.14` | |

**Key test — B9 (branch autonomy, requirement BR-5):**

1. On `BR1-R1`: `interface Serial0/0/0` → `shutdown`.
2. On a BR1 PC: `ipconfig /release` then `ipconfig /renew`.
3. **Expected:** the PC still receives an address from `BR1-STAFF-V110`, and can
   still ping `172.16.16.1` and other BR1 hosts. HQ and the Internet are
   unreachable, which is correct.
4. `no shutdown` the serial interface and confirm OSPF re-forms within ~40 s.

This is the test that proves the "local DHCP at branches" decision was worth the
extra configuration.

---

## Group C — Routing (OSPF)

| # | Device | Command | Expected result | Actual |
|---|---|---|---|---|
| C1 | HQ-R1 | `show ip ospf neighbor` | **3** neighbours in `FULL` state: 2.2.2.2, 3.3.3.3, 4.4.4.4 | |
| C2 | HQ-CORE | `show ip ospf neighbor` | 1 neighbour, 1.1.1.1, `FULL` | |
| C3 | HQ-R1 | `show ip ospf interface brief` | Gi0/1 in area 0, Se0/0/0 area 1, Se0/0/1 area 2, Lo0 area 0 | |
| C4 | any router | `show ip protocols` | `Reference bandwidth unit is 10000 mbps` — identical on all four routers | |
| C5 | HQ-CORE | `show ip route ospf` | `O IA 172.16.16.0/22` and `O IA 172.16.20.0/22` present; **no** more specific branch LAN prefixes (/27, /28, /29). The serial /30s and branch loopback /32s still appear on their own | |
| C6 | BR1-R1 | `show ip route` | `O*E2 0.0.0.0/0 [110/1] via 172.16.255.1`; `O IA` routes for HQ VLANs and for BR2 | |
| C7 | HQ-R1 | `show ip ospf database` | LSA type 1 in each area, type 3 summaries for the /22 ranges, type 5 for the default | |
| C8 | HQ-R1 | `show ip ospf` | reports `It is an area border router and autonomous system boundary router` | |

**C5 is the summarisation proof.** Screenshot it — it is the single most
quotable result in the report. Then, to demonstrate what summarisation *does*,
remove one `area 1 range` line on HQ-R1, re-run `show ip route ospf` on HQ-CORE,
count the routes, and put the two counts side by side.

**C9 — authentication is actually enforced:**
On `BR1-R1`, change the key: `interface Se0/0/0` → `ip ospf message-digest-key 1 md5 wrongkey`.
Expected: within 40 s the adjacency drops; `debug ip ospf adj` reports a
mismatched authentication key. Restore the correct key and confirm the neighbour
returns to `FULL`.

**C10 — convergence timing:**
On `HQ-R1`, shut `Se0/0/0`. From an HQ PC, run a continuous `ping 172.16.16.x`
before shutting the link. Record how many echoes are lost. Expected: the branch
becomes unreachable (there is no alternate path — that is the known single point
of failure), and the route is removed from the table within the OSPF dead
interval of 40 s. Confirm with `show ip route 172.16.16.0`.

---

## Group D — End-to-end connectivity matrix

`✓` = ping must succeed, `✗` = ping must fail (and failing is the *correct*
result, enforced by an ACL).

| From ↓ To → | HQ ENG PC | HQ Server .35 | HQ Guest PC | BR1 PC | BR2 PC | 192.0.2.53 | Device mgmt |
|---|---|---|---|---|---|---|---|
| HQ ENG PC | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ (mgmt ACL) |
| HQ Server .35 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ |
| **HQ Guest PC** | **✗** | **✗** | ✓ | **✗** | **✗** | **✓** | **✗** |
| BR1 PC | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ |
| **BR1 Guest PC** | **✗** | **✗** | ✗ | **✗** | **✗** | **✓** | **✗** |
| BR2 PC | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ |
| IT admin PC (VLAN 99) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **✓ (SSH)** |
| Outside host (198.51.100.10) | ✗ | ✓ only TCP 80/443 to 203.0.113.3 | ✗ | ✗ | ✗ | n/a | ✗ |

| # | Test | Expected | Actual |
|---|---|---|---|
| D1 | HQ ENG PC → `ping 172.16.1.35` | success | |
| D2 | HQ ENG PC → `ping 172.16.16.11` (BR1) | success | |
| D3 | BR1 PC → `ping 172.16.20.5` (BR2) | success, via HQ (verify with `tracert`) | |
| D4 | HQ Guest PC → `ping 172.16.1.35` | **fails** (`GUEST-VLAN40-IN` deny) | |
| D5 | HQ Guest PC → `ping 192.0.2.53` | success | |
| D6 | HQ Guest PC → `nslookup www.technova.local` | resolves (DNS permit rule) | |
| D7 | BR1 Guest PC → `ping 172.16.16.11` | **fails** | |
| D8 | BR1 Guest PC → `ping 192.0.2.53` | success | |
| D9 | BR1 PC → `tracert 172.16.20.5` | 3 hops: 172.16.16.1 → 172.16.255.1 → 172.16.20.5 | |

**D3/D9 note:** `tracert` output proves the hub-and-spoke path — branch-to-branch
traffic visibly transits HQ. This is worth a screenshot because it demonstrates
the topology decision in §2.2 actually behaving as designed.

---

## Group E — NAT and Internet

| # | Device | Command | Expected result | Actual |
|---|---|---|---|---|
| E1 | HQ PC | `ping 192.0.2.53` | success | |
| E2 | HQ-R1 | `show ip nat translations` | entries `icmp 203.0.113.2:x  172.16.0.x:x  192.0.2.53:x` | |
| E3 | HQ-R1 | `show ip nat statistics` | `Outside interfaces: Gi0/0`, `Inside interfaces: Gi0/1, Se0/0/0, Se0/0/1`; hits increasing, misses low | |
| E4 | BR2 PC | `ping 192.0.2.53` | success — proves `ip nat inside` is set on **both** serial interfaces | |
| E5 | external host 198.51.100.10 | browse `http://203.0.113.3` | web page served by 172.16.1.35 | |
| E6 | HQ-R1 | `show ip nat translations` (after E5) | static entry `--- 203.0.113.3  172.16.1.35` | |
| E7 | external host | `ping 203.0.113.3` | **fails** — `ISP-IN` permits only TCP 80/443 to that host | |
| E8 | external host | `ping 203.0.113.2` | **fails** — inbound echo-request is not permitted | |
| E9 | HQ-R1 | `show access-lists ISP-IN` | the `deny ip any any log` counter increments after E7/E8 | |

**E4 is the classic failure point.** If it fails while E1 succeeds, the cause is
almost always a missing `ip nat inside` on `Se0/0/1`.

---

## Group F — Security controls

| # | Test | How | Expected | Actual |
|---|---|---|---|---|
| F1 | Telnet is off | from IT admin PC: `telnet 172.16.255.241` | connection refused | |
| F2 | SSH works from mgmt | from IT admin PC (VLAN 99): `ssh -l netadmin 172.16.255.241` | login prompt, then privileged access | |
| F3 | SSH blocked elsewhere | from an ENG PC: `ssh -l netadmin 172.16.255.241` | connection refused / times out (`MGMT-ACCESS`) | |
| F4 | `deny any log` fires | HQ-R1: `show access-lists MGMT-ACCESS` after F3 | deny counter incremented | |
| F5 | Port security | move a PC to a different access port after sticky learning, or add a second PC via a hub | violation counter increments; `show port-security interface Fa0/x` shows it; traffic from the extra MAC is dropped | |
| F6 | Server port shutdown mode | connect a second device to HQ-ACC2 Fa0/20 | port goes `err-disabled` | |
| F7 | BPDU guard | connect a spare switch to an access port | port `err-disabled`; syslog `BPDUGUARD` message | |
| F8 | DHCP snooping | attach a second router/server configured as a DHCP server to an untrusted access port; renew a client | client still receives the **legitimate** address; `show ip dhcp snooping` shows drops on the untrusted port | |
| F9 | Native VLAN | HQ-CORE: `show interfaces trunk` | native VLAN 999 on every trunk; VLAN 1 absent from allowed lists | |
| F10 | Unused ports | any switch: `show interfaces status` | unused ports `disabled`, VLAN 999 | |
| F11 | OSPF auth | see test C9 | adjacency drops with a wrong key | |
| F12 | Passwords hashed | any device: `show running-config \| include secret\|password` | `enable secret 5 $...`; no cleartext line passwords | |

---

## Group G — Services

| # | Test | Expected | Actual |
|---|---|---|---|
| G1 | PC → `nslookup www.technova.local` | returns 172.16.1.35 | |
| G2 | PC browser → `http://www.technova.local` | page loads | |
| G3 | any device → `show ntp status` | `Clock is synchronized`, reference 172.16.255.241 | |
| G4 | syslog server (172.16.1.36) | receives messages from all 9 devices, with correct timestamps and source addresses | |
| G5 | trigger an event (shut/no shut an access port) then check the syslog server | `%LINK-3-UPDOWN` entry appears with the device name | |
| G6 | IP phone in VLAN 60 | receives an address from `HQ-VOICE-V60` and option 150 = 172.16.1.36 | |

---

## Summary sheet (fill in for the report)

| Group | Tests | Passed | Failed | Notes |
|---|---|---|---|---|
| A — Layer 2 | 8 | | | |
| B — DHCP | 8 | | | |
| C — Routing | 8 | | | |
| D — Connectivity | 9 | | | |
| E — NAT | 9 | | | |
| F — Security | 12 | | | |
| G — Services | 6 | | | |
| **Total** | **60** | | | |

## Screenshots to capture for the report

The minimum set that evidences the design claims, in this order:

1. Full Packet Tracer topology, all links green.
2. `show ip ospf neighbor` on HQ-R1 (3 × FULL).
3. `show ip route ospf` on HQ-CORE (the two /22 summaries) — **the key figure**.
4. `show etherchannel summary` on HQ-CORE (both bundles up).
5. `ipconfig` on one PC per site (DHCP working, including the branch).
6. `show ip nat translations` during an active Internet ping.
7. Guest PC: failed ping to a server **and** successful ping to 192.0.2.53, side by side.
8. External host loading `http://203.0.113.3` (inbound static NAT).
9. `show port-security interface` after a deliberate violation.
10. B9 branch-autonomy test: DHCP still working with the WAN link shut down.
