# 9. Build Guide — Cisco Packet Tracer

Follow this order. Building bottom-up (cable → VLANs → L2 → L3 → routing →
services → security) means each stage can be verified before the next one hides
its errors.

Tested against Packet Tracer 8.x.

---

## 9.1 Bill of materials

| Qty | Device type | Model in Packet Tracer | Modules to add |
|---|---|---|---|
| 4 | Router | **2911** | `HWIC-2T` in slot 0 — on HQ-R1, BR1-R1, BR2-R1 (ISP-R needs none) |
| 1 | Multilayer switch | **3560-24PS** | none |
| 4 | Switch | **2960-24TT** | none |
| 3 | Server | Server-PT | none |
| 6+ | PC | PC-PT | none |
| 2 | IP phone | IP Phone 7960 | needs a PoE port or a power adapter |
| 2–3 | Wireless AP | AccessPoint-PT | none |

**Adding the HWIC-2T:** click the router → *Physical* tab → **power the device
off** (the small switch on the chassis image) → drag `HWIC-2T` into slot 0 →
power back on. Skipping the power-off step is why the module "won't fit". This
module provides `Serial0/0/0` and `Serial0/0/1`.

## 9.2 Cabling table

| # | From | Port | To | Port | Cable |
|---|---|---|---|---|---|
| 1 | ISP-R | Gi0/0 | HQ-R1 | Gi0/0 | Copper straight-through |
| 2 | ISP-R | Gi0/1 | External-Server | Fa0 | Copper straight-through |
| 3 | HQ-R1 | Gi0/1 | HQ-CORE | Gi0/1 | Copper straight-through |
| 4 | HQ-R1 | **Se0/0/0** | BR1-R1 | Se0/0/0 | **Serial DCE** (drag from HQ-R1 first) |
| 5 | HQ-R1 | **Se0/0/1** | BR2-R1 | Se0/0/0 | **Serial DCE** (drag from HQ-R1 first) |
| 6 | HQ-CORE | Fa0/1 | HQ-ACC1 | Fa0/1 | Copper cross-over |
| 7 | HQ-CORE | Fa0/2 | HQ-ACC1 | Fa0/2 | Copper cross-over |
| 8 | HQ-CORE | Fa0/3 | HQ-ACC2 | Fa0/1 | Copper cross-over |
| 9 | HQ-CORE | Fa0/4 | HQ-ACC2 | Fa0/2 | Copper cross-over |
| 10 | HQ-ACC1 | Fa0/5 | PC-ENG1 | Fa0 | Copper straight-through |
| 11 | HQ-ACC1 | Fa0/10 | PC-SALES1 | Fa0 | Copper straight-through |
| 12 | HQ-ACC1 | Fa0/15 | IP-Phone-1 (then PC-ADMIN1 behind it) | Switch port | Copper straight-through |
| 13 | HQ-ACC2 | Fa0/5 | AP-GUEST-HQ | Port 0 | Copper straight-through |
| 14 | HQ-ACC2 | Fa0/20 | SRV-DNS (172.16.1.34) | Fa0 | Copper straight-through |
| 15 | HQ-ACC2 | Fa0/21 | SRV-WEB (172.16.1.35) | Fa0 | Copper straight-through |
| 16 | HQ-ACC2 | Fa0/22 | SRV-LOG (172.16.1.36) | Fa0 | Copper straight-through |
| 17 | HQ-ACC2 | Fa0/24 | PC-ITADMIN | Fa0 | Copper straight-through |
| 18 | BR1-R1 | Gi0/0 | BR1-SW1 | Fa0/1 | Copper straight-through |
| 19 | BR1-SW1 | Fa0/5 | PC-BR1-1 | Fa0 | Copper straight-through |
| 20 | BR1-SW1 | Fa0/20 | AP-GUEST-BR1 | Port 0 | Copper straight-through |
| 21 | BR2-R1 | Gi0/0 | BR2-SW1 | Fa0/1 | Copper straight-through |
| 22 | BR2-SW1 | Fa0/5 | PC-BR2-1 | Fa0 | Copper straight-through |
| 23 | BR2-SW1 | Fa0/20 | AP-GUEST-BR2 | Port 0 | Copper straight-through |

**On the serial DCE cable:** in Packet Tracer the end you click **first** becomes
the DCE end and shows a small clock icon. Always start at `HQ-R1`, because
`clock rate 2000000` is configured there. If you cable it the other way round the
line stays down with `Serial0/0/0 is down, line protocol is down` and
`show controllers` reports DTE on the wrong side.

**On cross-over vs straight-through:** modern Packet Tracer devices auto-MDIX, so
straight-through usually works switch-to-switch as well. Cross-over is specified
for the switch-to-switch links because it is the textbook-correct choice.

## 9.3 Build order

### Stage 1 — place and cable
Place all devices, apply the names from §9.1, and cable per §9.2. Expect some
link lights to stay amber (STP) or red (serial, before clocking) — that is
normal at this stage.

### Stage 2 — routers, base config
Paste, in this order: `ISP-R.txt`, `HQ-R1.txt`, `BR1-R1.txt`, `BR2-R1.txt`.

Verify after each: `show ip interface brief` — every configured interface
`up/up`. Fix a down serial link here, before routing is layered on top.

> **RSA key generation pauses for input.** When `crypto key generate rsa
> general-keys modulus 1024` runs it may ask for confirmation. Press Enter, then
> continue pasting the rest of the file. Paste the configuration in two chunks
> (before and after that line) if your terminal drops characters.

### Stage 3 — switches, base + VLANs + trunks
Paste `HQ-CORE.txt`, `HQ-ACC1.txt`, `HQ-ACC2.txt`, `BR1-SW1.txt`, `BR2-SW1.txt`.

Verify: `show vlan brief`, `show interfaces trunk`, and — critically —
`show etherchannel summary` on HQ-CORE and HQ-ACC1/2. Do not continue until both
bundles show `(SU)` with `(P)` members.

### Stage 4 — end devices
Set every PC and laptop to **DHCP** (Desktop → IP Configuration → DHCP).

Set the three servers **statically**:

| Server | IP | Mask | Gateway | DNS |
|---|---|---|---|---|
| SRV-DNS | 172.16.1.34 | 255.255.255.240 | 172.16.1.33 | 127.0.0.1 |
| SRV-WEB | 172.16.1.35 | 255.255.255.240 | 172.16.1.33 | 172.16.1.34 |
| SRV-LOG | 172.16.1.36 | 255.255.255.240 | 172.16.1.33 | 172.16.1.34 |
| External-Server | 198.51.100.10 | 255.255.255.0 | 198.51.100.1 | 198.51.100.10 |

Set `PC-ITADMIN` statically as well, so management access does not depend on
DHCP: `172.16.1.52 / 255.255.255.240 / gw 172.16.1.49`.

### Stage 5 — server services
On **SRV-DNS** → *Services* → **DNS** → On, and add:

| Name | Type | Address |
|---|---|---|
| `www.technova.local` | A Record | 172.16.1.35 |
| `syslog.technova.local` | A Record | 172.16.1.36 |

On **SRV-WEB** → *Services* → **HTTP** → On (edit `index.html` so the page is
easy to recognise in the NAT test).

On **SRV-LOG** → *Services* → **SYSLOG** → On, and **NTP** → On.

On **External-Server** → *Services* → **HTTP** → On, so there is something to
browse from inside.

### Stage 6 — verify routing
`show ip ospf neighbor` on HQ-R1 → three `FULL` neighbours. If one is missing:

| Symptom | Usual cause |
|---|---|
| No neighbour on a serial link | link down, or `bandwidth`/`clock rate` missing → check `show controllers` |
| Neighbour stuck in `INIT` or `EXSTART` | MD5 key mismatch, or `area X authentication message-digest` configured on one side only |
| Neighbour flaps | duplicated router-id (two devices with the same `router-id`) |
| Neighbour up but no routes | `passive-interface default` without the matching `no passive-interface` |

### Stage 7 — verify services and security
Work through the whole of §8 (docs/08). Capture the ten screenshots listed at the
end of that file as you go — retaking them later means rebuilding the same state.

## 9.4 Common build errors, and what they look like

| Symptom | Cause | Fix |
|---|---|---|
| PC gets `169.254.x.x` | DHCP not reaching the server | check `ip helper-address` on the SVI; check the access port VLAN; check `show ip dhcp pool` on HQ-R1 |
| PC gets an address from the **wrong** pool | the access port is in the wrong VLAN | `show interfaces status` and fix `switchport access vlan` |
| EtherChannel member shows `(I)` | mismatched settings between the two ends | compare `show run interface` on both; native VLAN and allowed list must match |
| `%CDP-4-NATIVE_VLAN_MISMATCH` | one trunk end still on native VLAN 1 | set `switchport trunk native vlan 999` on both ends |
| Branch pings HQ but not the Internet | `ip nat inside` missing on a serial interface | add it on HQ-R1 `Se0/0/0` **and** `Se0/0/1` |
| Guest can reach servers | ACL applied outbound, or on the wrong interface | it must be `ip access-group GUEST-VLAN40-IN in` on `interface Vlan40` |
| Guest cannot get an address | DHCP permits missing or placed after the deny line | the two `permit udp ... eq 67/68` lines must come **first** |
| SSH refused from the admin PC | admin PC not in VLAN 99, or wrong address | it must be inside `172.16.1.48/28` to pass `MGMT-ACCESS` |
| `show ip route` has no default at a branch | `default-information originate` missing on HQ-R1, or the static default is absent | both are required — OSPF only advertises a default it actually has |
| Access port `err-disabled` right after a PC is plugged in | BPDU guard fired, or a port-security violation in `shutdown` mode | `show interfaces status err-disabled`, then `shutdown` / `no shutdown` to recover |

## 9.5 Packet Tracer quirks in these configuration files

Four things that are correct IOS but behave slightly differently in the
simulator. None of them is an error in the design; each has a stated fallback.

**1. ACLs are applied to interfaces before they are defined.**
`ip access-group ISP-IN in` appears in §3 of `HQ-R1.txt`, while `ISP-IN` itself is
built in §6. Likewise `GUEST-VLAN40-IN` on `HQ-CORE`. This is intentional and
valid: IOS accepts a reference to an ACL that does not exist yet, and the ACL
becomes active the moment it is created a few lines later. Because each file is
pasted in one go, the window is milliseconds. If you prefer, paste the ACL
section first — the result is identical.

**2. `ip dhcp snooping trust` on a Port-channel.**
Real IOS accepts this on the logical `Po` interface. If your Packet Tracer build
rejects it there, apply it to the two physical member ports instead
(`interface range Fa0/1-2` → `ip dhcp snooping trust`). The effect is the same;
the trust boundary is the uplink either way.

**3. `port-channel load-balance src-dst-ip`.**
Supported on real 3560s; some Packet Tracer versions accept only a subset of
hashing options. If the command is rejected, drop it — the bundle still forms and
still forwards on both members, you just get the default MAC-based hash. Mention
the difference in the report rather than quietly removing the line.

**4. `crypto key generate rsa general-keys modulus 1024` pauses.**
It may prompt, and it takes a few seconds. Paste each configuration in two chunks
around that line if your terminal drops characters while the key is generated.

## 9.6 Saving the work

- `write memory` on every device (already the last line of each config file).
- **File → Save As** in Packet Tracer, named
  `TechNova-Network.pkt`, saved next to this repository.
- Keep the `.pkt` and the `configs/` folder together: if anyone opens the
  `.pkt` and something has drifted, the text files are the authoritative
  reference.
