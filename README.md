# 🏥 High-Availability Hospital Campus Network

A redundant, segmented enterprise campus network built in Cisco Packet Tracer, designed around a single guiding question:

> **What happens to a hospital's network — and its security policy — when a core router dies?**

This project answers that question with a working, tested implementation: dual-core routing with sub-second-aware gateway failover, loop-safe redundant switching, dynamic routing with a verified backup path, and department-level security segmentation that **survives a live failover**, not just a static config.

---

## 📐 Topology

```
                     [Core Router 1] ===== [Core Router 2]
                      (HSRP Active)   ×2    (HSRP Standby)
                      (OSPF + Loopback)     (OSPF + Loopback)
                         |                       |
                    [SW-DR1] ================ [SW-DR2]
                    (Distribution)   (redundant  (Distribution)
                         |            trunk link)     |
        ┌────────────────┼───────────────┼────────────────┐
        │                │               │                │
   [ADMIN SW]      [GUEST SW]      [ER/ICU SW]      [RADIOLOGY SW]
   80 hosts         100 hosts       40 hosts          25 hosts
   VLAN 20           VLAN 10         VLAN 30            VLAN 40
        │
   [Monitoring Server] (VLAN 50, single link to SW-DR1)
```

- **2 Core Routers**, directly interconnected by **two physical links** — a primary (OSPF-routed) and a backup (floating static route)
- **2 Distribution Switches** (SW-DR1, SW-DR2), cross-connected to every access switch **and** directly to each other, forming an intentional Layer 2 loop that Spanning Tree manages
- **4 Department Access Switches**, each dual-homed to both distribution switches for redundancy
- **1 isolated Monitoring VLAN**, single-homed by design (see [Design Decisions](#-design-decisions-and-why))

---

## 🧰 Technologies Implemented

| Layer | Technology | Purpose |
|---|---|---|
| Addressing | VLSM | Right-sized subnets for 5 VLANs + 2 router links, zero waste |
| Redundancy (L3 gateway) | HSRP | Sub-10-second automatic gateway failover, zero host reconfiguration |
| Redundancy (L2) | Rapid-PVST (802.1w) + PortFast | Loop prevention on redundant switch links; fast host-port startup |
| Redundancy (routing) | OSPF + Floating Static Route | Dynamic core routing with a verified automatic backup path |
| Segmentation | VLANs + 802.1Q Trunking | Department-level broadcast domain isolation |
| Access Control | Extended ACLs (5 rule sets) | Real security segmentation, not just connectivity |
| Access Control | Standard ACL + SSH | Management-plane security, keyed to a single trusted subnet |
| Services | DHCP (multi-pool) | Per-VLAN address assignment, gateway = HSRP virtual IP |

---

## 🧮 IP Addressing (VLSM)

| Segment | VLAN | CIDR | Network | Usable Range | Broadcast |
|---|---|---|---|---|---|
| Guest WiFi | 10 | /25 | 10.10.0.0 | .1 – .126 | .127 |
| Admin / Records | 20 | /25 | 10.10.0.128 | .129 – .254 | .255 |
| ER / ICU | 30 | /26 | 10.10.1.0 | .1 – .62 | .63 |
| Radiology | 40 | /27 | 10.10.1.64 | .65 – .94 | .95 |
| R1↔R2 (primary) | — | /30 | 10.10.2.0 | .1 – .2 | .3 |
| Monitoring | 50 | /29 | 10.10.2.8 | .9 – .14 | .15 |
| R1↔R2 (backup) | — | /30 | 10.10.2.16 | .17 – .18 | .19 |

Each VLAN's gateway is an **HSRP virtual IP** — never an individual router's real interface address — so that a DHCP-assigned default gateway remains valid regardless of which core router is currently active.

| VLAN | R1 (real) | R2 (real) | HSRP Virtual IP (gateway) |
|---|---|---|---|
| 10 – Guest | 10.10.0.1 | 10.10.0.2 | **10.10.0.3** |
| 20 – Admin | 10.10.0.129 | 10.10.0.130 | **10.10.0.131** |
| 30 – ER/ICU | 10.10.1.1 | 10.10.1.2 | **10.10.1.3** |
| 40 – Radiology | 10.10.1.65 | 10.10.1.66 | **10.10.1.67** |
| 50 – Monitoring | 10.10.2.9 | 10.10.2.10 | **10.10.2.11** |

---

## 🔐 Security Segmentation

The core security requirement: **isolate departments realistically, not uniformly.** Three distinct patterns were implemented, each solving a different access-control problem:

| Rule | Type | Behavior |
|---|---|---|
| ER/ICU ↔ Monitoring Server | Full two-way lockdown | ER/ICU can *only* reach the monitoring server; the server can *only* reach ER/ICU. Every other VLAN is explicitly denied from reaching the server. |
| Radiology → Admin/Records | One-way initiation | Radiology can initiate contact with Admin (e.g. submitting results); Admin cannot initiate contact with Radiology. Reply traffic is explicitly permitted via ICMP `echo-reply` matching. |
| Guest WiFi | Outbound-only isolation | Guest can reach outside the network but is denied from initiating to *any* internal department VLAN. |
| Management (SSH) | Source-restricted | Only the Admin/Records subnet may open an SSH session to either core router — enforced by a Standard ACL on the VTY lines, before the login prompt is even reached. |

All Extended ACLs are **replicated identically on both R1 and R2** and were verified, under a live simulated router failure, to continue enforcing correctly on whichever router is currently HSRP-active — see [Verification & Testing](#-verification--testing).

---

## 🌐 Routing Design

- **OSPF** runs *only* between R1 and R2's two dedicated interconnect links — deliberately kept off every VLAN-facing sub-interface via `passive-interface`, so OSPF adjacency reflects a genuine router-to-router relationship rather than an incidental one formed over shared LAN segments.
- **Stable Router IDs** are guaranteed via Loopback interfaces (`1.1.1.1` / `2.2.2.2`), independent of any physical interface's up/down state.
- **A floating static route** (AD 120, above OSPF's 110) rides the second physical R1↔R2 link, staying completely dormant until the primary OSPF-routed link fails — confirmed by watching the route disappear from and reappear in the routing table live during a link-shutdown test.

---

## ✅ Verification & Testing

Every claim in this README is backed by a `show` command or a live failure simulation, not just a working ping.

### 1. HSRP Gateway Failover
- Baseline: `show standby brief` on R1 shows **Active** for all 5 VLANs; R2 shows **Standby**.
- **R1's uplink shut down** → after HSRP's hold-time window, R2 correctly transitions to **Active** for all 5 VLANs.
- **R1 restored** → `preempt` reclaims Active status automatically, no manual intervention.

### 2. Security Policy Survives Failover
With R1 down and R2 serving as the active gateway for every VLAN, all 5 ACL rules were re-tested and held:
- ER/ICU → Monitoring: ✅ success
- Admin → Monitoring: ❌ blocked
- Guest → Admin: ❌ blocked
- Radiology → Admin: ✅ success
- Admin → Radiology: ❌ blocked
- Non-Admin SSH attempt to R2: ❌ connection refused

### 3. STP Loop Prevention
`show spanning-tree vlan 20` across the distribution and access layer confirms Rapid-PVST correctly identified the redundant DS1↔DS2 path and placed one port in **Alternate/Blocking** state, while all other paths remain forwarding.

### 4. Floating Static Route Failover
With the primary R1↔R2 link shut down:
- `show ip route` on R1 shows the OSPF-learned route to R2's loopback disappear
- The static route (AD 120) takes its place: `S  2.2.2.2 [120/0] via 10.10.2.18`
- `ping 2.2.2.2` continues to succeed throughout, confirming the backup path carries live traffic

---

## 🐛 Notable Debugging (Real Issues Found & Fixed)

Two bugs surfaced during testing that are worth documenting, since they reflect genuine troubleshooting rather than a scripted build:

**1. HSRP Hello packets silently blocked by their own department's ACL.**
The ER/ICU Extended ACL denied all traffic sourced from the ER/ICU subnet except toward the monitoring server — but R1 and R2's own real interface IPs on that VLAN *fall inside* that subnet's range, meaning their HSRP Hello multicasts were being filtered by the same rule meant for host traffic. Fixed by explicitly permitting UDP/1985 (HSRP) between the routers' real IPs, ahead of the broader deny rule.

**2. Asymmetric ACL logic breaking reply traffic.**
Early versions of the Monitoring-server and Guest-isolation ACLs blocked traffic by subnet in one direction only, which inadvertently caught *return* traffic (e.g. ping replies) sourced from the "blocked" side. Resolved by explicitly permitting ICMP `echo-reply` where legitimate two-way communication was required, while keeping new *inbound* connections (`echo`) denied — approximating stateful one-way access with stateless ACLs.

---

## 🛠️ Built With

- Cisco Packet Tracer 9.0
- Cisco IOS (2911 routers, 2960 switches)

## 📎 What This Project Demonstrates

VLSM subnetting · HSRP first-hop redundancy · Rapid-PVST spanning tree · 802.1Q trunking · OSPF (with passive-interface design) · floating static routing · multi-pool DHCP · Extended & Standard ACLs · SSH hardening · PortFast · live failure simulation and evidence-based verification

---

*This project was built and documented as a self-directed learning exercise in enterprise network design, redundancy, and security segmentation.*
