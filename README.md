# LDP-Based VPLS over MPLS/OSPF Core — Hands-On Lab (Arista EOS)

A hands-on lab demonstrating **Virtual Private LAN Service (VPLS)** with **LDP signaling** over an **MPLS-enabled OSPF core** using Arista EOS and containerlab.

Based on the LinkedIn article: *[Building VPLS (Full-mesh) Using LDP – Hands-On Lab - Part 1](https://www.linkedin.com/pulse/arista-eos-building-vpls-using-ldp-hands-on-lab-arun-ganesan-lmhvc/)* by **Arun Ganesan**.

---

## Overview

VPLS allows a service provider to extend Layer-2 connectivity across an MPLS core, making multiple geographically separated LAN sites behave like a single Ethernet broadcast domain. From the customer perspective, it operates like a normal bridged LAN.

### Key Points

- **Layer 2 VPN** solution over MPLS
- Emulates a virtual Ethernet segment across the MPLS core
- Uses **LDP** for pseudowire signaling (PWE3)
- Enables multiple customer sites to appear on the same broadcast domain (e.g., 172.16.10.0/24)

---

## Topology

```
                   LDP-Based VPLS over MPLS / OSPF Core
                   ====================================

                                       Lo0: 2.2.2.2/32
                                            P1
                                     +---------------+
                       |-------------|     Arista    |-----------|
                       |             +---------------+           |
                       |            Et1               Et2        |
                       |     10.0.10.0/30         10.0.30.0/30   |
                       |                                         |
                       | Et1                                     | Et1
                       |                                         |
+------------+       +---------------+                      +---------------+        +--------------+
|   CE1      |-------|      PE1      |                      |      PE2      |--------|   CE2        |
|172.16.10.10| VLAN10|   Lo0 1.1.1.1 |       VPLS VC        |   Lo0 4.4.4.4 | VLAN10 |172.16.10.20  |
|            |-------|     Arista    |     <========>       |     Arista    |--------|              |
+------------+ Eth8  +---------------+                      +---------------+  Eth8  +--------------+
                       |  Et2                                    |Et2
                       |                                         |
                  10.0.20.0/30                             10.0.40.0/30
                       |                                         |
                      Et1                                       Et2
                       |            +---------------+            |
                       |------------|      P2       |------------|
                                    |    Arista     |
                                    +---------------+
                                     Lo0: 3.3.3.3/32


          ════════  VPLS Pseudowire (LDP) PE-1 ↔ PE-2
          ────────  OSPF/LDP Control Plane
          --------  Data Plane
```

### Shared L2 Domain: `172.16.10.0/24`

---

## IP Addressing

| Device | Interface    | IP Address       | Description                |
|--------|-------------|------------------|----------------------------|
| CE-1   | Vlan10      | 172.16.10.10/24  | Customer LAN - Site A      |
| CE-2   | Vlan10      | 172.16.10.20/24  | Customer LAN - Site B      |
| PE-1   | Loopback0   | 1.1.1.1/32       | Router-ID / LDP Transport  |
| PE-1   | Ethernet1   | 10.0.10.1/30     | To P1                      |
| PE-1   | Ethernet2   | 10.0.20.1/30     | To P2                      |
| PE-1   | Ethernet8   | trunk (vlan 10)  | To CE-1                    |
| P1     | Loopback0   | 2.2.2.2/32       | Router-ID / LDP Transport  |
| P1     | Ethernet1   | 10.0.10.2/30     | To PE-1                    |
| P1     | Ethernet2   | 10.0.30.1/30     | To PE-2                    |
| P2     | Loopback0   | 3.3.3.3/32       | Router-ID / LDP Transport  |
| P2     | Ethernet1   | 10.0.20.2/30     | To PE-1                    |
| P2     | Ethernet2   | 10.0.40.1/30     | To PE-2                    |
| PE-2   | Loopback0   | 4.4.4.4/32       | Router-ID / LDP Transport  |
| PE-2   | Ethernet1   | 10.0.30.2/30     | To P1                      |
| PE-2   | Ethernet2   | 10.0.40.2/30     | To P2                      |
| PE-2   | Ethernet8   | trunk (vlan 10)  | To CE-2                    |

---

## Protocols

| Layer       | Protocol                     |
|-------------|------------------------------|
| Underlay    | OSPF (Area 0)                |
| MPLS        | LDP                          |
| L2 VPN      | VPLS — Manual Signaling (PWE3) |

---

## Prerequisites

- [containerlab](https://containerlab.dev/) installed
- Arista cEOS image imported (`ceos:latest`)
- Docker / OrbStack running

---

## Deployment

### Deploy the lab

```bash
sudo containerlab deploy -t topology.yaml
```

### Destroy the lab

```bash
sudo containerlab destroy -t topology.yaml
```

### Access a node

```bash
docker exec -it clab-ldp-vpls-PE-1 Cli
```

---

## Verification Commands

### 1. OSPF Adjacency

```
PE-1# show ip ospf neighbor
PE-1# show ip route ospf
```

### 2. MPLS / LDP

```
PE-1# show mpls ldp neighbor
PE-1# show mpls ldp bindings
PE-1# show mpls route
```

### 3. VPLS / Pseudowire

```
PE-1# show patch panel detail
PE-1# show mpls ldp bindings neighbor 4.4.4.4
```

### 4. End-to-End L2 Connectivity

From CE-1, ping CE-2 (both on 172.16.10.0/24):

```
CE-1# ping 172.16.10.20 source 172.16.10.10
```

### 5. MAC Learning Across VPLS

```
PE-1# show mac address-table
CE-1# show mac address-table
```

---

## Repo Structure

```
LDP-VPLS/
├── README.md            # This file
├── topology.yaml        # Containerlab topology
├── configs/
│   ├── CE-1.cfg         # Customer Edge 1 (Site A)
│   ├── PE-1.cfg         # Provider Edge 1 (VPLS + LDP)
│   ├── P1.cfg           # Provider Core 1 (Transit LSR)
│   ├── P2.cfg           # Provider Core 2 (Transit LSR)
│   ├── PE-2.cfg         # Provider Edge 2 (VPLS + LDP)
│   └── CE-2.cfg         # Customer Edge 2 (Site B)
└── images/
    └── topology.png     # Network topology diagram
```

---

## References

- [RFC 4762 — VPLS Using LDP Signaling](https://datatracker.ietf.org/doc/html/rfc4762)
- [RFC 4447 — Pseudowire Setup and Maintenance Using LDP](https://datatracker.ietf.org/doc/html/rfc4447)
- [Arista EOS Product Documentation](https://www.arista.com/en/support/product-documentation)

---

## Author

**Arun Ganesan** — Technical Solutions Engineer at Arista Networks
