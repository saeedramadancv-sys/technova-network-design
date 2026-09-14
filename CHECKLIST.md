# Submission checklist

Work down the list. Anything unticked is either a gap in the build or something
to declare explicitly in the report.

## A. Documents

- [ ] `docs/10-final-report.md` — title page filled in (name, ID, instructor, date)
- [ ] §8.2 results table filled in with actual pass/fail per group
- [ ] Screenshots inserted (see the list of 10 at the end of `docs/08-...`)
- [ ] Report exported to the format your course requires (PDF or Word)
- [ ] Diagrams rendered to images and embedded (paste the `.mmd` files into
      https://mermaid.live and export PNG/SVG)

## B. Packet Tracer build

- [ ] All 9 devices placed and named exactly as in `docs/09-...` §9.1
- [ ] `HWIC-2T` installed in HQ-R1, BR1-R1, BR2-R1 (device powered off first)
- [ ] All 23 cables per the cabling table; serial cables started at HQ-R1 (DCE)
- [ ] Every config from `configs/` pasted and `write memory` done
- [ ] Servers statically addressed; DNS, HTTP, SYSLOG, NTP services enabled
- [ ] PCs set to DHCP; PC-ITADMIN set statically in VLAN 99
- [ ] `.pkt` file saved as `TechNova-Network-<yourname>.pkt`

## C. The verifications that must pass before you submit

Ten checks that cover the whole design. If all ten pass, the build is sound.

- [ ] `show ip ospf neighbor` on HQ-R1 → **3 neighbours, all FULL**
- [ ] `show ip route ospf` on HQ-CORE → `172.16.16.0/22` and `172.16.20.0/22`
      as `O IA`, and **no** more specific branch prefixes
- [ ] `show etherchannel summary` on HQ-CORE → `Po1(SU)` and `Po2(SU)`, members `(P)`
- [ ] `show interfaces trunk` on every switch → native VLAN **999**, pruned lists
- [ ] A PC in each of the three sites gets a correct DHCP address
- [ ] HQ PC and BR2 PC can both `ping 192.0.2.53` (proves PAT on both serials)
- [ ] Guest PC **cannot** ping `172.16.1.35` but **can** ping `192.0.2.53`
- [ ] External host loads `http://203.0.113.3` but **cannot** ping it
- [ ] SSH works from PC-ITADMIN, fails from an Engineering PC
- [ ] Branch-autonomy test B9: serial shut → branch DHCP still works

## D. Consistency audit (catches the errors markers actually find)

- [ ] Every gateway address in the configs matches `docs/03-...` §3.7 exactly
- [ ] Every subnet mask matches its prefix (see §3.8 table)
- [ ] The same OSPF MD5 key on **both** ends of every link
- [ ] `auto-cost reference-bandwidth 10000` on **all four** routers
- [ ] No duplicate OSPF `router-id`
- [ ] Allowed-VLAN list identical on both ends of each trunk
- [ ] EtherChannel member ports identically configured on both ends
- [ ] `ip helper-address 172.16.255.14` on HQ SVIs 10, 20, 30, 40, 60
- [ ] `ip nat inside` on Gi0/1 **and** Se0/0/0 **and** Se0/0/1 of HQ-R1
- [ ] Guest ACLs: the two DHCP permits come **before** the internal deny
- [ ] No device still has a default hostname or an unencrypted password

## E. Things to be ready to say out loud

- [ ] Why OSPF, not EIGRP or RIP
- [ ] Why areas, when a single area would work at this size
- [ ] What `area range` achieved, with the before/after number
- [ ] Why branch DHCP is local rather than relayed to HQ
- [ ] Why the native VLAN is 999
- [ ] Why the guest ACL is ordered the way it is
- [ ] The six limitations in report §9.2, and the fix for each

## F. Optional additions, if you have time and want extra marks

Each is a self-contained addition that does not disturb what already works:

- [ ] **Dynamic ARP Inspection** — `ip arp inspection vlan 10,20,30,40,60` on the
      access switches. DHCP snooping (its prerequisite) is already configured, so
      this is genuinely two lines, and it closes the ARP-spoofing gap named in
      §9.2 item 5.
- [ ] **IPsec site-to-site VPN** HQ↔BR1 — addresses §9.2 item 3 and is the single
      most valuable addition in security terms.
- [ ] **Second HQ router with HSRP** on the core link — addresses the single point
      of failure in §9.2 item 2. The largest of these additions.
- [ ] **IPv6 dual-stack** on one site, e.g. `2001:db8:acad::/48` with OSPFv3 —
      demonstrates that the design is not IPv4-locked.
- [ ] **QoS for voice** — `mls qos` on HQ-CORE, trust the phone's CoS, priority
      queue for VLAN 60. Natural next step now that voice has its own VLAN.
