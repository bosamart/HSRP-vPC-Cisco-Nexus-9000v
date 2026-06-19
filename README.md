# HSRP + vPC on Cisco Nexus 9000v — Production-Style Reference

A lab build of First-Hop Redundancy (HSRP) on a pair of Cisco Nexus 9000v
switches, configured the way it's done in a real datacenter: an **HSRP gateway
running on top of a vPC pair**, with a **dedicated routed keepalive link**.

Built and tested in EVE-NG on Proxmox.

---

## Table of Contents

- [Key Concept: HSRP vs vPC vs Keepalive](#key-concept-hsrp-vs-vpc-vs-keepalive)
- [Topology](#topology)
- [IP / VLAN Plan](#ip--vlan-plan)
- [Configuration — N9K-1 (Primary)](#configuration--n9k-1-primary)
- [Configuration — N9K-2 (Secondary)](#configuration--n9k-2-secondary)
- [vPC Member Ports (MLAG) — Dual-Homed Devices](#vpc-member-ports-mlag--dual-homed-devices)
- [Verification](#verification)
- [Design Notes](#design-notes)
- [HSRP vs VRRP](#hsrp-vs-vrrp)
- [Peer-Keepalive: mgmt0 vs Dedicated Port](#peer-keepalive-mgmt0-vs-dedicated-port)

---

## Key Concept: HSRP vs vPC vs Keepalive

Three independent control mechanisms run on the same pair of switches. They are
often confused — they are **not** the same thing.

| Mechanism | Job | Runs over | Dedicated link? |
|-----------|-----|-----------|-----------------|
| **HSRP / VRRP (FHRP)** | Elect the active L3 gateway (virtual IP) | The data VLAN / SVI itself | No |
| **vPC peer-link** | L2 multipathing + state sync (MAC, ARP, IGMP) | A dedicated port-channel | Yes |
| **vPC peer-keepalive** | Detect a dead peer / prevent split-brain | mgmt0 *or* a dedicated routed port | Yes |

**Important:** HSRP (and VRRP) need **no** dedicated heartbeat link. Their
"heartbeat" is the hello packets, which ride on the shared data subnet. The
peer-link and keepalive belong to **vPC**, a separate feature. A plain HSRP setup
with no vPC needs neither.

With vPC, HSRP forwards **active/active in the data plane**: the control plane
elects one Active and one Standby, but *both* peers program the virtual MAC in
hardware and route locally. Traffic never hairpins across the peer-link just to be
routed — so aggressive sub-second timers are unnecessary.

---

## Topology

```
        ┌──────────┐   peer-link (Po1, Eth1/1-2)   ┌──────────┐
        │  N9K-1   │═══════════════════════════════│  N9K-2   │
        │ (vPC pri)│                               │ (vPC sec)│
        │          │── keepalive (Eth1/3, /30) ────│          │
        │ Po10/vpc10│                              │ Po10/vpc10│
        └────┬─────┘                               └─────┬────┘
       Eth1/10│                                          │Eth1/10
              └──────────────┐         ┌─────────────────┘
                           ┌─┴─────────┴─┐
                           │  Access SW  │  <- one LACP port-channel
                           └──────┬──────┘     (split across both chassis)
                                  │
                              Hosts (VLAN 10/20/30)
```

- **Peer-link** — Po1 (Eth1/1-2), trunk carrying data VLANs + vPC control plane.
- **Keepalive** — Eth1/3, dedicated routed P2P link in its own VRF (heartbeat only).
- **vPC member port (MLAG)** — Po10 (Eth1/10), dual-homed trunk to the access switch.
- **Uplink** — Eth1/49, routed uplink to core (tracked for HSRP failover).

---

## IP / VLAN Plan

| VLAN | Purpose  | Subnet         | VIP (gateway) | N9K-1 IP | N9K-2 IP | HSRP grp |
|------|----------|----------------|---------------|----------|----------|----------|
| 10   | Servers  | 10.10.10.0/24  | 10.10.10.1    | .2       | .3       | 10       |
| 20   | Users    | 10.10.20.0/24  | 10.10.20.1    | .2       | .3       | 20       |
| 30   | Mgmt     | 10.10.30.0/24  | 10.10.30.1    | .2       | .3       | 30       |

| Function | N9K-1 | N9K-2 |
|----------|-------|-------|
| Keepalive (Eth1/3, VRF `vPC-KEEPALIVE`) | 10.99.99.1/30 | 10.99.99.2/30 |
| vPC role priority | 100 (primary) | 200 (secondary) |
| HSRP priority | 110 + preempt (Active) | 100 default (Standby) |

> Matching the HSRP group number to the VLAN ID is for readability only — the
> group is locally significant per SVI.

---

## Configuration — N9K-1 (Primary)

```
feature interface-vlan
feature hsrp
feature lacp
feature vpc

! --- HSRP authentication ---
key chain HSRP-KEY
  key 1
    key-string Str0ngHSRPkey!

! --- Dedicated keepalive VRF + link ---
vrf context vPC-KEEPALIVE

interface Ethernet1/3
  description vPC-PEER-KEEPALIVE
  no switchport
  vrf member vPC-KEEPALIVE
  ip address 10.99.99.1/30
  no shutdown

! --- vPC domain ---
vpc domain 1
  role priority 100
  peer-switch
  peer-keepalive destination 10.99.99.2 source 10.99.99.1 vrf vPC-KEEPALIVE
  peer-gateway
  ip arp synchronize
  ipv6 nd synchronize
  delay restore 150
  auto-recovery

! --- Peer-link ---
interface port-channel1
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,30
  spanning-tree port type network
  vpc peer-link

interface Ethernet1/1-2
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,30
  channel-group 1 mode active
  no shutdown

vlan 10,20,30

! --- Uplink tracking ---
track 1 interface Ethernet1/49 line-protocol

! --- SVIs + HSRP ---
interface Vlan10
  no shutdown
  description SERVERS-GW
  ip address 10.10.10.2/24
  hsrp version 2
  hsrp 10
    authentication md5 key-chain HSRP-KEY
    preempt delay minimum 180
    priority 110
    track 1 decrement 20
    ip 10.10.10.1

interface Vlan20
  no shutdown
  description USERS-GW
  ip address 10.10.20.2/24
  hsrp version 2
  hsrp 20
    authentication md5 key-chain HSRP-KEY
    preempt delay minimum 180
    priority 110
    track 1 decrement 20
    ip 10.10.20.1

interface Vlan30
  no shutdown
  description MGMT-GW
  ip address 10.10.30.2/24
  hsrp version 2
  hsrp 30
    authentication md5 key-chain HSRP-KEY
    preempt delay minimum 180
    priority 110
    track 1 decrement 20
    ip 10.10.30.1
```

---

## Configuration — N9K-2 (Secondary)

Identical except: keepalive source/dest swapped, `role priority 200`, SVI IPs end
in `.3`, and **no priority/preempt/track** on the SVIs (stays at default 100, so
N9K-1 wins the HSRP election).

```
feature interface-vlan
feature hsrp
feature lacp
feature vpc

key chain HSRP-KEY
  key 1
    key-string Str0ngHSRPkey!

vrf context vPC-KEEPALIVE

interface Ethernet1/3
  description vPC-PEER-KEEPALIVE
  no switchport
  vrf member vPC-KEEPALIVE
  ip address 10.99.99.2/30
  no shutdown

vpc domain 1
  role priority 200
  peer-switch
  peer-keepalive destination 10.99.99.1 source 10.99.99.2 vrf vPC-KEEPALIVE
  peer-gateway
  ip arp synchronize
  ipv6 nd synchronize
  delay restore 150
  auto-recovery

interface port-channel1
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,30
  spanning-tree port type network
  vpc peer-link

interface Ethernet1/1-2
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,30
  channel-group 1 mode active
  no shutdown

vlan 10,20,30

interface Vlan10
  no shutdown
  description SERVERS-GW
  ip address 10.10.10.3/24
  hsrp version 2
  hsrp 10
    authentication md5 key-chain HSRP-KEY
    ip 10.10.10.1

interface Vlan20
  no shutdown
  description USERS-GW
  ip address 10.10.20.3/24
  hsrp version 2
  hsrp 20
    authentication md5 key-chain HSRP-KEY
    ip 10.10.20.1

interface Vlan30
  no shutdown
  description MGMT-GW
  ip address 10.10.30.3/24
  hsrp version 2
  hsrp 30
    authentication md5 key-chain HSRP-KEY
    ip 10.10.30.1
```

---

## vPC Member Ports (MLAG) — Dual-Homed Devices

On Nexus, **MLAG = vPC**. The peer-link + keepalive + vPC domain you configured
above *is* the MLAG control plane. A **vPC member port** is the downstream LAG that
is split across both switches but appears as a single LACP port-channel to the
device connected below — that device has no idea (and needs no idea) that its two
links land on different chassis.

Nothing special is added to HSRP for this. The member port is a Layer-2 trunk
carrying the same VLANs that have HSRP SVIs. The dual-homed device points its
default gateway at the HSRP VIP, and thanks to vPC active/active forwarding,
whichever physical link the device hashes onto, that Nexus routes the traffic
locally — no hairpin across the peer-link.

**The rule:** both peers configure a port-channel with the **same `vpc <id>`**. That
shared ID tells the pair "these two local port-channels are the same downstream
LAG."

### On the Nexus pair (identical on N9K-1 and N9K-2)

```
interface port-channel10
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,30
  spanning-tree port type edge trunk
  vpc 10

interface Ethernet1/10
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,30
  channel-group 10 mode active
  no shutdown
```

- The **`vpc 10`** ID must match on both peers. The port-channel *number* need not
  equal the vPC ID, but matching them (Po10 / vpc 10) is standard convention.
- Use **LACP** (`mode active`) toward the device.
- `spanning-tree port type edge trunk` only for hosts/servers; for a real switch
  downstream, use a normal trunk (`port type network` if it's another bridge).
- Configure on **both** peers — if only one side has `vpc 10`, it won't form a vPC.

### On the downstream device (plain LACP — example IOS access switch)

```
interface range Gi0/1 - 2
  channel-group 1 mode active

interface port-channel1
  switchport mode trunk
  switchport trunk allowed vlan 10,20,30
```

### Consistency (the usual gotcha)

A vPC member port-channel must be **consistent across both peers** — allowed VLANs,
mode, speed, MTU, STP type, etc. **Type-1** mismatches *suspend* the vPC; **Type-2**
mismatches only warn.

```
show vpc consistency-parameters vpc 10
```

### Orphan ports (single-homed devices)

Anything connected to only one Nexus is an **orphan port**; its traffic may cross
the peer-link and can be isolated during certain failures. To make orphans go down
when the peer-link drops:

```
interface Ethernet1/20
  vpc orphan-port suspend
```

### Verifying the MLAG port

```
show vpc                          ! vPC10 listed, status "up", consistency "success"
show port-channel summary         ! Po10 up, member ports bundled (P)
show vpc consistency-parameters vpc 10
show mac address-table vlan 10    ! host MAC learned on the vPC
```

From a host below: `ping 10.10.10.1` (the HSRP VIP) should succeed regardless of
which uplink it hashes onto.

---

## Verification

```
show vpc                       ! peer-link up, "peer adjacency formed ok"
show vpc role
show vpc peer-keepalive        ! "peer is alive", correct source/dest/VRF
show hsrp brief                ! one Active (P) per group, correct virtual IP
show hsrp group 10
show hsrp detail               ! confirm auth + vMAC (grp 10 -> 0000.0c9f.f00a)
show track brief
```

**Expected results**

- `show hsrp brief` — N9K-1 = **Active**, N9K-2 = **Standby** for every group;
  state `P` (preempt) on N9K-1.
- `show vpc peer-keepalive` — `peer is alive`.
- HSRPv2 vMAC follows `0000.0C9F.Fxxx`, where `xxx` = group number in hex
  (group 10 -> `...f00a`). Handy for ARP troubleshooting.

**Failover test**

1. `show hsrp brief` — confirm N9K-1 is Active.
2. Shut N9K-1's uplink: `interface Eth1/49 ; shutdown`.
3. Track object goes Down -> HSRP priority drops 110 - 20 = 90 (< 100).
4. N9K-2 becomes Active. Restore the uplink; after `preempt delay minimum 180`,
   N9K-1 reclaims Active.

---

## Design Notes

- **Don't over-tune timers under vPC.** Forwarding is already active/active, so the
  defaults (hello 3 / hold 10) are fine. Tuning to `timers 1 3` is harmless but
  rarely needed here.
- **`peer-gateway` matters.** Some servers, NAS, and load balancers reply to the
  burned-in router MAC instead of the HSRP vMAC. `peer-gateway` lets each peer
  route for the other's MAC so those packets aren't dropped. NX-OS auto-disables
  IP redirects on those SVIs when enabled — expected behaviour.
- **`preempt delay minimum`** stops the primary from grabbing Active the instant it
  reboots, before uplinks/routing converge. 180s is a safe starting point.
- **Tracking decrement** must drop priority *below* the standby's: 110 - 20 = 90 <
  100, so failover triggers. A decrement of 5 would never fail over.
- **`peer-switch`** presents a single STP root across both peers — clean for
  production, but both peers must have matching STP priority/config. Add it only
  once the pair is stable.

---

## HSRP vs VRRP

Both are FHRP and work the same way (virtual IP + virtual MAC, one active forwarder,
hellos on the shared subnet). Swapping HSRP for VRRP would not change the link
layout — VRRP adds no heartbeat link either.

| | HSRP | VRRP |
|--|------|------|
| Standard | Cisco proprietary | Open (RFC 5798) |
| Roles | Active / Standby | Master / Backup |
| Hello dest | 224.0.0.102 (v2), UDP 1985 | 224.0.0.18, IP protocol 112 |
| Virtual IP | Separate from interface IPs | Can be the real interface IP |
| NX-OS feature | `feature hsrp` | `feature vrrp` / `feature vrrpv3` |

> Note: FHRP is **not** like firewall HA (Palo Alto HA1/HA2, FortiGate HA). Those
> use dedicated physical heartbeat + session-sync links. FHRP has no equivalent.

---

## Peer-Keepalive: mgmt0 vs Dedicated Port

Both are valid, production-supported options.

| | mgmt0 (management VRF) | Dedicated routed port |
|--|------------------------|-----------------------|
| Best when | You have an OOB management switch | No OOB network / full control wanted |
| EVE-NG | Needs a management cloud/bridge | Easier — just a direct link |
| Isolation | Management VRF | Custom VRF (e.g. `vPC-KEEPALIVE`) |

**Two hard rules for a dedicated port:**

1. **Put it in its own VRF** — keep keepalive out of the global routing table.
2. **It must be a separate path from the peer-link** — so you can distinguish
   "peer is dead" from "peer-link is down." Never run keepalive over the peer-link
   or any vPC.

Other points:

- Make it a plain routed P2P link (`no switchport`, /30 or /31).
- Run **no** routing protocol on that VRF — it's just the connected subnet.
- A single dedicated link is acceptable: keepalive only matters at the instant the
  peer-link fails. Bundle two ports into a port-channel only if you want the most
  paranoid design.
- Keepalive carries tiny hellos only — it can be your smallest/slowest port. The
  **peer-link** is the one that needs bandwidth.

---

*Lab environment: EVE-NG on Proxmox, Cisco Nexus 9000v (NX-OS).*
