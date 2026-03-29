# SIIT-DC Analysis: IPv6-Only Datacenter with Jool Translation

## Executive Summary

This document analyzes the feasibility and benefits of adopting the **SIIT-DC (Stateless IP/ICMP Translation for Data Centers)** model using **Jool** for the CDN-POP project. The proposal is to transform the entire datacenter infrastructure to **IPv6-only**, adopting an **IPv6-first** approach and translating IPv4 traffic at the network edge.

---

## Table of Contents

1. [Introduction to SIIT-DC](#1-introduction-to-siit-dc)
2. [Current Architecture Analysis](#2-current-architecture-analysis)
3. [Proposed SIIT-DC Architecture](#3-proposed-siit-dc-architecture)
4. [Jool Implementation Details](#4-jool-implementation-details)
5. [Component Adaptation](#5-component-adaptation)
6. [Comparative Analysis](#6-comparative-analysis)
7. [Implementation Roadmap](#7-implementation-roadmap)
8. [Conclusion and Recommendations](#8-conclusion-and-recommendations)

---

## 1. Introduction to SIIT-DC

### 1.1 What is SIIT-DC?

**SIIT-DC** (RFC 7756, RFC 6145) is an IETF standard that defines how to operate a datacenter with IPv6-only infrastructure while maintaining connectivity with IPv4-only clients on the internet. The key concept is **stateless translation** at the network edge.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SIIT-DC CONCEPT                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   IPv4-Only          Stateless            IPv6-Only                          │
│   Internet  ───────▶ Translator ◀───────  Datacenter                          │
│   Clients            (Jool)               Infrastructure                      │
│                                                                              │
│   ┌─────────┐       ┌─────────────┐       ┌─────────────────────────────┐    │
│   │ Client  │       │   Jool      │       │  IPv6-Only Servers          │    │
│   │ IPv4    │ ─────▶│  SIIT-DC    │──────▶│  - Native IPv6 applications │    │
│   │ Only    │       │  Translator │       │  - Simpler network config   │    │
│   └─────────┘       └─────────────┘       │  - No dual-stack complexity │    │
│        │                  │               └─────────────────────────────┘    │
│        │                  │                           │                      │
│        │                  │                           │                      │
│        │◀─────────────────┘◀──────────────────────────┘                      │
│              Response (IPv4 translated back)                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Key RFCs

| RFC | Title | Description |
|-----|-------|-------------|
| **RFC 6145** | IP/ICMP Translation Algorithm | Defines stateless IPv4/IPv6 translation |
| **RFC 7756** | SIIT-DC: Stateless Translation | Datacenter-specific SIIT deployment |
| **RFC 6052** | IPv6 Addressing of IPv4/IPv6 Translators | Address translation format |
| **RFC 6791** | ICMPv6 to ICMPv4 Translation | ICMP message translation |

### 1.3 What is Jool?

**Jool** is an open-source, Linux kernel module implementation of SIIT-DC and NAT64. It provides:

- **Stateless translation (SIIT)** - One-to-one IPv4/IPv6 address mapping
- **Stateful translation (NAT64)** - Many-to-one with port translation
- **EAM (Explicit Address Mapping)** - Custom IPv4-to-IPv6 mappings
- **High performance** - Kernel-level packet processing
- **Production ready** - Used in production by multiple organizations

```
┌─────────────────────────────────────────────────────────────────┐
│                    JOOL FEATURES                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Translation Modes                        │ │
│  │                                                             │ │
│  │  1. SIIT (Stateless)                                        │ │
│  │     - 1:1 IPv4 ↔ IPv6 mapping                               │ │
│  │     - No state tracking                                     │ │
│  │     - Predictable address mapping                           │ │
│  │                                                             │ │
│  │  2. NAT64 (Stateful)                                        │ │
│  │     - N:1 IPv4 ↔ IPv6 mapping                               │ │
│  │     - Port-based translation                                │ │
│  │     - Connection tracking required                          │ │
│  │                                                             │ │
│  │  3. EAM (Explicit Address Mapping)                          │ │
│  │     - Custom IPv4 → IPv6 mappings                           │ │
│  │     - Flexible address assignment                           │ │
│  │     - Ideal for server infrastructure                       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Performance:                                                    │
│  - Kernel module (not userspace)                                │
│  - ~10 Mpps on modern hardware                                  │
│  - Low latency overhead                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Current Architecture Analysis

### 2.1 Current Dual-Stack Complexity

The current architecture requires maintaining **dual-stack (IPv4 + IPv6)** throughout the entire infrastructure:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CURRENT DUAL-STACK COMPLEXITY                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Component          IPv4 Required?    IPv6 Required?    Complexity          │
│  ─────────────────────────────────────────────────────────────────────────  │
│  Katran LB          YES              YES               Dual XDP programs    │
│  Silverton          YES              YES               ARP + NDP handling   │
│  BIRD               YES              YES               Dual BGP sessions    │
│  Traffic Steering   YES              YES               Dual IPSets          │
│  Edge Servers       YES              YES               Dual IP config       │
│  IPIP Tunnels       YES              YES               Dual tunnel modes    │
│  SNAT Pools         YES              YES               Dual NAT rules       │
│                                                                              │
│  Total Complexity: HIGH                                                      │
│  - 2x configuration for every component                                      │
│  - 2x troubleshooting paths                                                  │
│  - 2x potential failure points                                               │
│  - Protocol-specific bugs in either stack                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Current Pain Points

| Pain Point | Description | Impact |
|------------|-------------|--------|
| **Dual BPF Maps** | Katran needs separate VIP/Real maps for IPv4 and IPv6 | 2x memory, 2x code complexity |
| **Dual Neighbor Protocols** | Silverton must handle ARP (IPv4) and NDP (IPv6) | 2x protocol handlers |
| **Dual BGP Sessions** | Separate eBGP/iBGP for each address family | 2x BGP configuration |
| **Dual IPIP Tunnels** | ipip (IPv4) and ip6ip6 (IPv6) tunnel interfaces | 2x tunnel management |
| **Dual SNAT Pools** | Separate IPv4 and IPv6 SNAT configurations | 2x NAT rules |
| **MTU Issues** | Different MTU requirements for IPv4 vs IPv6 encapsulation | Operational complexity |
| **Address Planning** | Must maintain both IPv4 and IPv6 addressing schemes | 2x IPAM |

### 2.3 IPv4 Address Scarcity

The current architecture still relies on IPv4 addresses for:

- VIP addresses (anycast)
- Real server addresses
- SNAT pools
- Tunnel endpoints

This creates:
- **Cost**: IPv4 addresses are expensive (~$30-50/IP)
- **Limitations**: Limited pool constrains scaling
- **Complexity**: NAT/overhead to conserve addresses

---

## 3. Proposed SIIT-DC Architecture

### 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SIIT-DC ARCHITECTURE                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         INTERNET                                             │
│              ┌──────────────────────────────┐                               │
│              │  IPv4 Clients  │ IPv6 Clients │                               │
│              └───────┬───────┴──────┬───────┘                               │
│                      │                │                                      │
│                      ▼                ▼                                      │
│              ┌───────────────────────────────┐                               │
│              │      BORDER SWITCHES          │                               │
│              │  (Arista - Dual-Stack Edge)   │                               │
│              │                               │                               │
│              │  IPv4 VIP: 203.0.113.10       │                               │
│              │  IPv6 VIP: 2001:db8::10       │                               │
│              └───────┬───────────┬───────────┘                               │
│                      │           │                                           │
│          ┌───────────┘           └──────────┐                               │
│          ▼                                    ▼                               │
│  ┌───────────────────┐              ┌───────────────────┐                   │
│  │   JOOL SIIT-DC    │              │   JOOL SIIT-DC    │                   │
│  │   Translator 1    │              │   Translator 2    │                   │
│  │                   │              │                   │                   │
│  │  IPv4 → IPv6      │              │  IPv4 → IPv6      │                   │
│  │  IPv6 → IPv4      │              │  IPv6 → IPv4      │                   │
│  │  (Stateless)      │              │  (Stateless)      │                   │
│  └─────────┬─────────┘              └─────────┬─────────┘                   │
│            │                                  │                              │
│            └──────────────┬───────────────────┘                              │
│                           │                                                  │
│                           ▼                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    IPv6-ONLY DATACENTER                               │   │
│  │                                                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │                    KATRAN LOAD BALANCERS                        │ │   │
│  │  │                                                                 │ │   │
│  │  │  ┌──────────────────┐              ┌──────────────────┐         │ │   │
│  │  │  │   Katran LB-1    │              │   Katran LB-2    │         │ │   │
│  │  │  │   XDP/eBPF       │              │   XDP/eBPF       │         │ │   │
│  │  │  │   IPv6-ONLY      │              │   IPv6-ONLY      │         │ │   │
│  │  │  │   VIP: 2001:db8::10            │   VIP: 2001:db8::10        │ │   │
│  │  │  └────────┬─────────┘              └────────┬─────────┘         │ │   │
│  │  └───────────┼─────────────────────────────────┼───────────────────┘ │   │
│  │              │         ip6ip6 Encap            │                     │   │
│  │              └────────────────┬────────────────┘                     │   │
│  │                               │                                      │   │
│  │  ┌────────────────────────────▼────────────────────────────────────┐ │   │
│  │  │                    EDGE SERVERS (IPv6-ONLY)                      │ │   │
│  │  │                                                                  │ │   │
│  │  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐         │ │   │
│  │  │  │Edge-1  │ │Edge-2  │ │Edge-3  │ │Edge-4  │ │Edge-5  │         │ │   │
│  │  │  │2001:   │ │2001:   │ │2001:   │ │2001:   │ │2001:   │         │ │   │
│  │  │  │db8:2::1│ │db8:2::2│ │db8:2::3│ │db8:2::4│ │db8:2::5│         │ │   │
│  │  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘         │ │   │
│  │  │                                                                  │ │   │
│  │  │  Each server:                                                    │ │   │
│  │  │  - IPv6 address only (no IPv4 configured)                        │ │   │
│  │  │  - VIP on loopback (IPv6)                                        │ │   │
│  │  │  - ip6ip6 tunnel interface                                       │ │   │
│  │  │  - BIRD for IPv6 route reflection                                │ │   │
│  │  └──────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                       │   │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │   │
│  │  │                    MANAGEMENT / CONTROL PLANE                     │ │   │
│  │  │                                                                   │ │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │ │   │
│  │  │  │  Silverton   │  │  Healthcheck │  │  Monitoring  │            │ │   │
│  │  │  │  (NDP only)  │  │  Service     │  │  (Prometheus)│            │ │   │
│  │  │  └──────────────┘  └──────────────┘  └──────────────┘            │ │   │
│  │  └──────────────────────────────────────────────────────────────────┘ │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Translation Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    IPv4 CLIENT TO IPv6 SERVER FLOW                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. IPv4 CLIENT REQUEST                                                      │
│     ┌─────────────────────────────────────────────────────────────────────┐ │
│     │  Client (203.0.113.100) sends SYN to VIP 203.0.113.10:443           │ │
│     │                                                                      │ │
│     │  Packet:                                                             │ │
│     │    src: 203.0.113.100:45678                                         │ │
│     │    dst: 203.0.113.10:443                                            │ │
│     │    proto: TCP                                                        │ │
│     └─────────────────────────────────────────────────────────────────────┘ │
│                              │                                               │
│                              ▼                                               │
│  2. JOOL SIIT-DC TRANSLATION (Stateless)                                     │
│     ┌─────────────────────────────────────────────────────────────────────┐ │
│     │  Jool translates IPv4 to IPv6 using RFC 6052 mapping:               │ │
│     │                                                                      │ │
│     │  IPv4 → IPv6 Translation:                                            │ │
│     │    src: 203.0.113.100 → 2001:db8:0:0:0:0:cb00:7164                  │ │
│     │         (RFC 6052: IPv4-embedded IPv6 address)                       │ │
│     │                                                                      │ │
│     │    dst: 203.0.113.10 → 2001:db8::10                                 │ │
│     │         (EAM mapping: VIP IPv4 → VIP IPv6)                          │ │
│     │                                                                      │ │
│     │  Translated Packet:                                                  │ │
│     │    src: 2001:db8::cb00:7164:45678                                   │ │
│     │    dst: 2001:db8::10:443                                            │ │
│     │    proto: TCP                                                        │ │
│     └─────────────────────────────────────────────────────────────────────┘ │
│                              │                                               │
│                              ▼                                               │
│  3. KATRAN IPv6-ONLY PROCESSING                                              │
│     ┌─────────────────────────────────────────────────────────────────────┐ │
│     │  Katran receives IPv6 packet:                                        │ │
│     │                                                                      │ │
│     │  a) VIP lookup: 2001:db8::10 → found in vip_map_v6                  │ │
│     │  b) Maglev hash: hash(src_ip, src_port, dst_ip, dst_port)           │ │
│     │  c) Select real server: 2001:db8:2::3                               │ │
│     │  d) ip6ip6 encapsulation:                                            │ │
│     │                                                                      │ │
│     │  Encapsulated Packet:                                                │ │
│     │    Outer src: 2001:db8:1::1 (RSS-friendly)                          │ │
│     │    Outer dst: 2001:db8:2::3 (Real server)                           │ │
│     │    Inner src: 2001:db8::cb00:7164:45678                             │ │
│     │    Inner dst: 2001:db8::10:443                                      │ │
│     └─────────────────────────────────────────────────────────────────────┘ │
│                              │                                               │
│                              ▼                                               │
│  4. EDGE SERVER PROCESSING (IPv6-ONLY)                                       │
│     ┌─────────────────────────────────────────────────────────────────────┐ │
│     │  Edge server receives ip6ip6 packet:                                │ │
│     │                                                                      │ │
│     │  a) Decapsulate: remove outer IPv6 header                           │ │
│     │  b) VIP on loopback: 2001:db8::10 → accept packet                   │ │
│     │  c) Application processes request                                    │ │
│     │  d) Response sent back to 2001:db8::cb00:7164                       │ │
│     └─────────────────────────────────────────────────────────────────────┘ │
│                              │                                               │
│                              ▼                                               │
│  5. RESPONSE PATH (IPv6 → IPv4)                                              │
│     ┌─────────────────────────────────────────────────────────────────────┐ │
│     │  Server response:                                                    │ │
│     │    src: 2001:db8::10:443                                            │ │
│     │    dst: 2001:db8::cb00:7164:45678                                   │ │
│     │                                                                      │ │
│     │  Jool reverse translation:                                           │ │
│     │    src: 2001:db8::10 → 203.0.113.10                                 │ │
│     │    dst: 2001:db8::cb00:7164 → 203.0.113.100                         │ │
│     │                                                                      │ │
│     │  Response to client:                                                 │ │
│     │    src: 203.0.113.10:443                                            │ │
│     │    dst: 203.0.113.100:45678                                         │ │
│     └─────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 IPv6 Client Flow (No Translation)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    IPv6 CLIENT TO IPv6 SERVER FLOW                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  IPv6 clients connect directly - NO translation needed!                      │
│                                                                              │
│  1. IPv6 CLIENT REQUEST                                                      │
│     ┌─────────────────────────────────────────────────────────────────────┐ │
│     │  Client (2001:db8:100::50) sends SYN to VIP 2001:db8::10:443        │ │
│     │                                                                      │ │
│     │  Packet:                                                             │ │
│     │    src: 2001:db8:100::50:45678                                      │ │
│     │    dst: 2001:db8::10:443                                            │ │
│     │    proto: TCP                                                        │ │
│     └─────────────────────────────────────────────────────────────────────┘ │
│                              │                                               │
│                              ▼                                               │
│  2. DIRECT TO KATRAN (No Translation!)                                       │
│     ┌─────────────────────────────────────────────────────────────────────┐ │
│     │  Packet passes through Jool unchanged:                               │ │
│     │                                                                      │ │
│     │  Jool rule: IPv6 → IPv6 = pass through                               │ │
│     │                                                                      │ │
│     │  Packet unchanged:                                                   │ │
│     │    src: 2001:db8:100::50:45678                                      │ │
│     │    dst: 2001:db8::10:443                                            │ │
│     └─────────────────────────────────────────────────────────────────────┘ │
│                              │                                               │
│                              ▼                                               │
│  3. KATRAN PROCESSING (Same as before)                                       │
│     ┌─────────────────────────────────────────────────────────────────────┐ │
│     │  Normal IPv6 processing:                                             │ │
│     │  - VIP lookup                                                        │ │
│     │  - Maglev hash                                                       │ │
│     │  - ip6ip6 encapsulation                                              │ │
│     │  - Forward to real server                                            │ │
│     └─────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  BENEFIT: IPv6 traffic has zero translation overhead!                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Jool Implementation Details

### 4.1 Jool Installation

```bash
# Ubuntu/Debian
apt-get update
apt-get install jool-dkms jool-tools

# Or build from source for latest features
git clone https://github.com/NICMx/Jool.git
cd Jool
./configure
make
make install
modprobe jool_siit
```

### 4.2 Jool SIIT-DC Configuration

```bash
# /etc/jool/siit-config.json
{
    "instance": "siit-dc-pop",
    "framework": "netfilter",
    
    "global": {
        "pool6": "2001:db8::/96",
        "pool6791v4": "192.0.0.4/30",
        "disable-filtering": true,
        "zeroize-traffic-class": true,
        "override-tos": true,
        "tos": 0,
        "mtu-plateaus": [1280, 1480, 88, 1500]
    },
    
    "eam": [
        # VIP Mappings (IPv4 → IPv6)
        {"ipv4": "203.0.113.10", "ipv6": "2001:db8::10"},
        {"ipv4": "203.0.113.11", "ipv6": "2001:db8::11"},
        {"ipv4": "203.0.113.12", "ipv6": "2001:db8::12"},
        
        # Real Server Mappings (if needed for health checks)
        {"ipv4": "10.0.2.1", "ipv6": "2001:db8:2::1"},
        {"ipv4": "10.0.2.2", "ipv6": "2001:db8:2::2"},
        {"ipv4": "10.0.2.3", "ipv6": "2001:db8:2::3"},
        {"ipv4": "10.0.2.4", "ipv6": "2001:db8:2::4"},
        {"ipv4": "10.0.2.5", "ipv6": "2001:db8:2::5"},
        {"ipv4": "10.0.2.6", "ipv6": "2001:db8:2::6"}
    ],
    
    "pool4": [
        # IPv4 addresses used for translation
        {
            "protocol": "TCP",
            "prefix": "203.0.113.10/32",
            "port-range": "1-65535"
        },
        {
            "protocol": "UDP",
            "prefix": "203.0.113.10/32",
            "port-range": "1-65535"
        },
        {
            "protocol": "ICMP",
            "prefix": "203.0.113.10/32"
        }
    ]
}
```

### 4.3 Jool Instance Setup

```bash
#!/bin/bash
# setup-jool-siit.sh

# Load kernel module
modprobe jool_siit

# Create Jool instance
jool_siit instance add "siit-dc-pop" --netfilter

# Configure global settings
jool_siit --instance "siit-dc-pop" update global \
    pool6 "2001:db8::/96" \
    disable-filtering true

# Add EAM mappings for VIPs
jool_siit --instance "siit-dc-pop" eamt add \
    203.0.113.10 2001:db8::10

jool_siit --instance "siit-dc-pop" eamt add \
    203.0.113.11 2001:db8::11

jool_siit --instance "siit-dc-pop" eamt add \
    203.0.113.12 2001:db8::12

# Add EAM mappings for real servers (optional, for health checks)
for i in {1..6}; do
    jool_siit --instance "siit-dc-pop" eamt add \
        10.0.2.$i 2001:db8:2::$i
done

# Display configuration
jool_siit --instance "siit-dc-pop" eamt display

# Enable instance
jool_siit --instance "siit-dc-pop" enable
```

### 4.4 RFC 6052 Address Mapping

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RFC 6052 IPv4-EMBEDDED IPv6 ADDRESS FORMAT                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Format:  prefix:ipv4-address::suffix                                       │
│                                                                              │
│  Example with /96 prefix (2001:db8::/96):                                   │
│                                                                              │
│  IPv4 Address: 203.0.113.100                                                │
│  Hex: cb00:7164                                                             │
│                                                                              │
│  IPv6 Representation:                                                        │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  2001:0db8:0000:0000:0000:0000:cb00:7164                          │    │
│  │  │<--- prefix --->│<---- zeros ---->│<-- IPv4 hex -->│              │    │
│  │       /96              32 bits           32 bits                    │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Short form: 2001:db8::cb00:7164                                            │
│                                                                              │
│  Benefits:                                                                   │
│  - Stateless: IPv4 address is embedded in IPv6                              │
│  - Reversible: Can extract original IPv4 from IPv6                          │
│  - No state table needed for translation                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.5 EAM (Explicit Address Mapping)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EAM - EXPLICIT ADDRESS MAPPING                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  EAM allows custom IPv4 → IPv6 mappings (not RFC 6052 embedded)             │
│                                                                              │
│  Use Cases:                                                                  │
│  1. VIP Mapping: Map public IPv4 VIP to internal IPv6 VIP                   │
│  2. Server Mapping: Map IPv4 real server IPs to IPv6 addresses              │
│  3. Service Mapping: Map specific services to specific IPv6 addresses       │
│                                                                              │
│  Example:                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  EAM Entry:                                                          │   │
│  │    IPv4: 203.0.113.10 (Public VIP)                                   │   │
│  │    IPv6: 2001:db8::10 (Internal VIP)                                 │   │
│  │                                                                      │   │
│  │  Translation:                                                        │   │
│  │    IPv4 packet to 203.0.113.10 → IPv6 packet to 2001:db8::10        │   │
│  │    IPv6 packet from 2001:db8::10 → IPv4 packet from 203.0.113.10    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Priority: EAM takes precedence over RFC 6052 pool6 mapping                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Component Adaptation

### 5.1 Katran - IPv6-Only Simplification

With SIIT-DC, Katran becomes **IPv6-only**, dramatically simplifying the codebase:

```c
// BEFORE: Dual-stack XDP (complex)
SEC("xdp")
int xdp_load_balancer(struct xdp_md *ctx) {
    u16 eth_proto = bpf_ntohs(eth->h_proto);
    
    switch (eth_proto) {
    case ETH_P_IP:    // 0x0800 - IPv4
        return process_ipv4(ctx, data, data_end);  // Complex IPv4 path
    case ETH_P_IPV6:  // 0x86DD - IPv6
        return process_ipv6(ctx, data, data_end);  // IPv6 path
    default:
        return XDP_PASS;
    }
}

// AFTER: IPv6-only XDP (simple)
SEC("xdp")
int xdp_load_balancer(struct xdp_md *ctx) {
    u16 eth_proto = bpf_ntohs(eth->h_proto);
    
    // Only process IPv6 - IPv4 translated by Jool before reaching Katran
    if (eth_proto != ETH_P_IPV6)
        return XDP_PASS;
    
    return process_ipv6(ctx, data, data_end);
}
```

**Benefits:**
- Single BPF map for VIPs (IPv6 only)
- Single BPF map for real servers (IPv6 only)
- Single LRU cache structure
- Single encapsulation path (ip6ip6)
- ~50% code reduction in XDP layer

### 5.2 Katran Configuration (IPv6-Only)

```cpp
// KatranConfig - Simplified for IPv6-only
struct KatranConfig {
    std::string mainInterface;      // "eth0"
    std::string v6TunInterface;     // "ipip60" - Only tunnel needed
    std::string balancerProgPath;   // path to balancer.bpf.o
    std::string healthcheckingProgPath;
    std::vector<uint8_t> defaultMac;
    uint32_t priority = 10;
    uint32_t maxVips = 64;
    uint32_t maxReals = 128;
    uint32_t chRingSize = 65537;
    uint64_t LruSize = 8000000;
    // No more v4TunInterface!
    // No more IPv4-specific settings!
};
```

```bash
# VIP Configuration - IPv6 only
./katran_goclient -A -t [2001:db8::10]:443

# Real servers - IPv6 only
./katran_goclient -a -t [2001:db8::10]:443 -r 2001:db8:2::1 -w 10
./katran_goclient -a -t [2001:db8::10]:443 -r 2001:db8:2::2 -w 10
./katran_goclient -a -t [2001:db8::10]:443 -r 2001:db8:2::3 -w 10
./katran_goclient -a -t [2001:db8::10]:443 -r 2001:db8:2::4 -w 10
./katran_goclient -a -t [2001:db8::10]:443 -r 2001:db8:2::5 -w 10
./katran_goclient -a -t [2001:db8::10]:443 -r 2001:db8:2::6 -w 10
```

### 5.3 Silverton - NDP-Only Simplification

With SIIT-DC, Silverton only needs to handle **NDP (IPv6 Neighbor Discovery Protocol)**:

```cpp
// BEFORE: Dual-stack neighbor handling
class SilvertonController : public eos::sdk_agent {
    eos::arp_table_mgr *arp_mgr_;        // IPv4 ARP
    eos::neighbor_table_mgr *neighbor_mgr_;  // IPv6 NDP
    
    void on_arp_entry_set(...);   // ARP handler
    void on_neighbor_entry_set(...);  // NDP handler
};

// AFTER: IPv6-only neighbor handling
class SilvertonController : public eos::sdk_agent {
    eos::neighbor_table_mgr *neighbor_mgr_;  // Only NDP needed!
    
    void on_neighbor_entry_set(...);  // Only NDP handler
    // No ARP handling!
};
```

**gRPC Protocol Simplification:**

```protobuf
// BEFORE: Dual-stack protocol
message NeighborUpdate {
    string ip_address = 1;
    string mac_address = 2;
    string interface = 3;
    int32 vrf = 4;
    UpdateType type = 5;
    AddressFamily family = 6;  // Need to specify IPv4 or IPv6
}

// AFTER: IPv6-only protocol
message NeighborUpdate {
    string ipv6_address = 1;  // Always IPv6
    string mac_address = 2;
    string interface = 3;
    int32 vrf = 4;
    UpdateType type = 5;
    // No address family needed!
}
```

### 5.4 BIRD - IPv6-Only BGP

```bash
# /etc/bird/bird.conf - Simplified IPv6-only

# eBGP with transit providers (IPv6 only)
protocol bgp transit1_v6 {
    local as 64512;
    neighbor 2001:db8:0:1::1 as 64513;
    import all;
    export all;
    
    ipv6 {
        import all;
        export all;
    };
}

protocol bgp transit2_v6 {
    local as 64512;
    neighbor 2001:db8:0:2::1 as 64514;
    import all;
    export all;
    
    ipv6 {
        import all;
        export all;
    };
}

# iBGP route reflection to hosts (IPv6 only)
protocol bgp host_rr_v6 {
    local as 64512;
    neighbor 2001:db8:2::1 as 64512;
    neighbor 2001:db8:2::2 as 64512;
    neighbor 2001:db8:2::3 as 64512;
    neighbor 2001:db8:2::4 as 64512;
    neighbor 2001:db8:2::5 as 64512;
    neighbor 2001:db8:2::6 as 64512;
    
    import none;
    export all;
    rr client;
    
    ipv6 {
        import none;
        export all;
    };
}

# Kernel route export (IPv6 only)
protocol kernel {
    scan time 60;
    import none;
    export all;
    
    ipv6 {
        import none;
        export all;
    };
}

# Static routes for VIPs (IPv6 only)
protocol static {
    ipv6;
    route 2001:db8::10/128 via "lo";
    route 2001:db8::11/128 via "lo";
    route 2001:db8::12/128 via "lo";
}
```

### 5.5 Traffic Steering - IPv6-Only

```yaml
# /etc/steerd/config.yaml - Simplified IPv6-only

paths:
  - id: 0
    name: "primary"
    interface: "eth0"
    ipset_v6: "egress_path_0_v6"  # Only IPv6 ipset needed
    fwmark: 0x100
    snat_pool_v6:
      - "2001:db8:100::2"
      - "2001:db8:100::3"
      - "2001:db8:100::4"
      - "2001:db8:100::5"
      - "2001:db8:100::6"
      - "2001:db8:100::7"

  - id: 1
    name: "failover"
    interface: "eth1"
    ipset_v6: "egress_path_1_v6"
    fwmark: 0x101
    snat_pool_v6:
      - "2001:db8:200::2"
      - "2001:db8:200::3"
      - "2001:db8:200::4"
      - "2001:db8:200::5"
      - "2001:db8:200::6"
      - "2001:db8:200::7"
```

```bash
# /etc/nftables/traffic-steering.nft - Simplified IPv6-only

table inet traffic_steering {
    # IPv6 destination sets only
    set egress_path_0_v6 {
        type ipv6addr
        size 65536
    }
    
    set egress_path_1_v6 {
        type ipv6addr
        size 65536
    }
    
    chain output {
        type route hook output priority mangle; policy accept;
        
        # IPv6 marking only
        ip6 daddr @egress_path_0_v6 meta mark set 0x100
        ip6 daddr @egress_path_1_v6 meta mark set 0x101
    }
    
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        
        # IPv6 SNAT only
        oif "eth0" meta mark 0x100 snat ip6 to 2001:db8:100::2-2001:db8:100::7
        oif "eth1" meta mark 0x101 snat ip6 to 2001:db8:200::2-2001:db8:200::7
    }
}
```

### 5.6 Edge Server Configuration (IPv6-Only)

```bash
#!/bin/bash
# setup-edge-server-ipv6-only.sh

# Create ip6ip6 tunnel interface (only one needed!)
ip link add name ipip60 type ip6tnl mode ip6ip6 external
ip link set up dev ipip60

# Configure IPv6 address on main interface
ip addr add 2001:db8:2::1/64 dev eth0

# Configure VIP on loopback (IPv6 only)
ip addr add 2001:db8::10/128 dev lo

# No IPv4 configuration needed!
# No IPv4 ARP handling!
# No IPv4 rp_filter settings!

# Increase MTU for ip6ip6 overhead
ip link set dev eth0 mtu 9000

# Disable IPv6 autoconfiguration (optional)
sysctl -w net.ipv6.conf.eth0.autoconf=0
sysctl -w net.ipv6.conf.eth0.accept_ra=0

# Enable IPv6 forwarding
sysctl -w net.ipv6.conf.all.forwarding=1
```

---

## 6. Comparative Analysis

### 6.1 Complexity Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLEXITY COMPARISON                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Component          Dual-Stack          SIIT-DC          Reduction          │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  Katran XDP         2 parsers           1 parser         50%                │
│  BPF Maps           4 maps              2 maps           50%                │
│  Encapsulation      2 modes             1 mode           50%                │
│                                                                              │
│  Silverton          ARP + NDP           NDP only         50%                │
│  gRPC Protocol      Dual-family         Single-family    30%                │
│                                                                              │
│  BIRD               4 sessions          2 sessions       50%                │
│  Route Tables       2 tables            1 table          50%                │
│                                                                              │
│  Traffic Steering   Dual IPSets         Single IPSet     50%                │
│  SNAT Rules         Dual pools          Single pool      50%                │
│                                                                              │
│  Server Config      Dual IP             Single IP        50%                │
│  Tunnel Interfaces  2 tunnels           1 tunnel         50%                │
│                                                                              │
│  ─────────────────────────────────────────────────────────────────────────  │
│  OVERALL COMPLEXITY REDUCTION: ~50%                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Performance Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE COMPARISON                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Metric                    Dual-Stack       SIIT-DC        Notes            │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  IPv4 Client PPS           ~10 Mpps         ~8 Mpps        Jool overhead    │
│  IPv6 Client PPS           ~9 Mpps          ~10 Mpps       No translation!  │
│                                                                              │
│  Latency (IPv4 client)     ~0.5 µs          ~1.0 µs        Translation cost │
│  Latency (IPv6 client)     ~0.6 µs          ~0.5 µs        Optimized path   │
│                                                                              │
│  Memory (BPF maps)         ~48 MB           ~24 MB         50% reduction    │
│  Memory (BIRD routes)      ~2 GB            ~1 GB          50% reduction    │
│                                                                              │
│  CPU (XDP processing)      2 cores          1 core         50% reduction    │
│  CPU (Translation)         0                1 core         Jool overhead    │
│                                                                              │
│  ─────────────────────────────────────────────────────────────────────────  │
│  KEY INSIGHT:                                                               │
│  - IPv6 clients get BETTER performance (no translation)                     │
│  - IPv4 clients have slight overhead (stateless translation)                │
│  - Overall infrastructure is simpler and more efficient                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Cost Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COST COMPARISON                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Item                    Dual-Stack       SIIT-DC        Savings            │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  IPv4 Addresses (VIPs)   10 IPs           10 IPs         $0 (still needed)  │
│  IPv4 Addresses (SNAT)   60 IPs           0 IPs          $1,800-3,000/yr    │
│  IPv4 Addresses (Real)   6 IPs            0 IPs          $180-300/yr        │
│                                                                              │
│  Development Time        100%             60%            40% time savings    │
│  Testing Time            100%             60%            40% time savings    │
│  Troubleshooting         100%             50%            50% time savings    │
│                                                                              │
│  Operational Complexity  High             Medium         Significant        │
│  Training Required       Dual-stack       IPv6 + Jool    Simpler           │
│                                                                              │
│  ─────────────────────────────────────────────────────────────────────────  │
│  ESTIMATED ANNUAL SAVINGS: $2,000-3,300 + 40% operational overhead          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.4 Pros and Cons Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PROS AND CONS                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PROS (SIIT-DC with Jool)                                                   │
│  ─────────────────────────────────────────────────────────────────────────  │
│  ✅ 50% reduction in infrastructure complexity                              │
│  ✅ Single protocol to manage (IPv6)                                        │
│  ✅ No IPv4 address scarcity for internal infrastructure                     │
│  ✅ IPv6 clients get native performance (no translation)                    │
│  ✅ Stateless translation = no connection tracking overhead                  │
│  ✅ Simpler troubleshooting (single protocol)                                │
│  ✅ Future-proof (IPv6-first architecture)                                   │
│  ✅ Reduced BGP configuration                                                │
│  ✅ Reduced BPF map memory usage                                             │
│  ✅ Easier to scale (no IPv4 limitations)                                    │
│                                                                              │
│  CONS (SIIT-DC with Jool)                                                   │
│  ─────────────────────────────────────────────────────────────────────────  │
│  ❌ Additional component (Jool) to manage                                    │
│  ❌ Slight latency overhead for IPv4 clients (~0.5 µs)                      │
│  ❌ Need to maintain IPv4 VIPs for external access                           │
│  ❌ Learning curve for Jool configuration                                    │
│  ❌ ICMP translation complexity (need to handle ICMP errors)                 │
│  ❌ MTU considerations (IPv6 minimum 1280 vs IPv4 576)                       │
│  ❌ Some applications may have IPv4 hardcoded assumptions                     │
│  ❌ Need to ensure origin servers support IPv6                               │
│                                                                              │
│  NEUTRAL                                                                    │
│  ─────────────────────────────────────────────────────────────────────────  │
│  ⚖️ IPv4 clients still work (transparent translation)                       │
│  ⚖️ Same external VIPs advertised via BGP                                   │
│  ⚖️ Same anycast behavior for global load balancing                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Implementation Roadmap

### 7.1 Phase 1: Jool Deployment (Weeks 1-2)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 1: JOOL DEPLOYMENT                                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Objectives:                                                                 │
│  - Deploy Jool SIIT-DC translators                                          │
│  - Configure EAM mappings for VIPs                                          │
│  - Test IPv4 → IPv6 translation                                              │
│                                                                              │
│  Tasks:                                                                      │
│  1. Install Jool on dedicated translation servers                           │
│  2. Configure EAM mappings: IPv4 VIPs → IPv6 VIPs                          │
│  3. Configure RFC 6052 pool for client addresses                            │
│  4. Test translation with synthetic traffic                                 │
│  5. Verify ICMP translation works correctly                                 │
│                                                                              │
│  Success Criteria:                                                           │
│  - IPv4 client can reach IPv6 server through Jool                           │
│  - Translation is stateless (no connection table)                           │
│  - ICMP errors are translated correctly                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Phase 2: IPv6-Only Katran (Weeks 3-4)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 2: IPv6-ONLY KATRAN                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Objectives:                                                                 │
│  - Simplify Katran to IPv6-only operation                                   │
│  - Remove IPv4 code paths                                                   │
│  - Test with IPv6 traffic                                                   │
│                                                                              │
│  Tasks:                                                                      │
│  1. Create IPv6-only BPF maps (vip_map_v6, real_map_v6)                     │
│  2. Remove IPv4 parsing code from XDP program                               │
│  3. Configure IPv6 VIPs and real servers                                    │
│  4. Test ip6ip6 encapsulation                                               │
│  5. Verify Maglev hashing works with IPv6                                   │
│                                                                              │
│  Success Criteria:                                                           │
│  - Katran processes IPv6 traffic correctly                                  │
│  - ip6ip6 encapsulation works                                               │
│  - DSR mode functions with IPv6                                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 Phase 3: IPv6-Only Silverton (Weeks 5-6)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 3: IPv6-ONLY SILVERTON                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Objectives:                                                                 │
│  - Simplify Silverton to NDP-only operation                                 │
│  - Remove ARP handling code                                                 │
│  - Test neighbor propagation                                                │
│                                                                              │
│  Tasks:                                                                      │
│  1. Update Silverton Controller to monitor NDP only                         │
│  2. Update Silverton Agent to handle IPv6 neighbors only                    │
│  3. Simplify gRPC protocol (remove address family field)                    │
│  4. Test NDP propagation from switch to servers                             │
│  5. Verify static neighbor entries are created                              │
│                                                                              │
│  Success Criteria:                                                           │
│  - NDP entries propagated correctly                                         │
│  - Servers can reach transit gateway via IPv6                               │
│  - No ARP handling needed                                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.4 Phase 4: IPv6-Only BIRD (Weeks 7-8)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 4: IPv6-ONLY BIRD                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Objectives:                                                                 │
│  - Configure IPv6-only BGP sessions                                         │
│  - Remove IPv4 BGP configuration                                            │
│  - Test route reflection                                                    │
│                                                                              │
│  Tasks:                                                                      │
│  1. Configure IPv6 eBGP sessions with transits                              │
│  2. Configure IPv6 iBGP route reflection                                    │
│  3. Remove IPv4 BGP sessions                                                │
│  4. Test full IPv6 routing table propagation                                │
│  5. Verify routes installed in kernel                                       │
│                                                                              │
│  Success Criteria:                                                           │
│  - Full IPv6 routing table in server kernels                                │
│  - IPv6 routes reflected to all hosts                                       │
│  - No IPv4 BGP sessions needed                                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.5 Phase 5: Traffic Steering IPv6-Only (Weeks 9-10)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 5: TRAFFIC STEERING IPv6-ONLY                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Objectives:                                                                 │
│  - Simplify traffic steering to IPv6-only                                   │
│  - Remove IPv4 IPSets and SNAT rules                                        │
│  - Test egress path selection                                               │
│                                                                              │
│  Tasks:                                                                      │
│  1. Update steerd config for IPv6-only paths                                │
│  2. Remove IPv4 IPSets from nftables                                        │
│  3. Configure IPv6 SNAT pools                                               │
│  4. Test egress path failover                                               │
│  5. Verify IPv6 SNAT works correctly                                        │
│                                                                              │
│  Success Criteria:                                                           │
│  - Traffic steering works with IPv6 only                                    │
│  - Failover between paths works                                             │
│  - IPv6 SNAT functions correctly                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.6 Phase 6: Integration Testing (Weeks 11-12)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 6: INTEGRATION TESTING                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Objectives:                                                                 │
│  - End-to-end testing of complete SIIT-DC architecture                      │
│  - Performance benchmarking                                                 │
│  - Failover testing                                                         │
│                                                                              │
│  Tasks:                                                                      │
│  1. Test IPv4 client → IPv6 server flow                                     │
│  2. Test IPv6 client → IPv6 server flow                                     │
│  3. Benchmark PPS and latency                                               │
│  4. Test Jool translator failover                                           │
│  5. Test Katran LB failover                                                 │
│  6. Test egress path failover                                               │
│                                                                              │
│  Success Criteria:                                                           │
│  - All traffic flows work correctly                                         │
│  - Performance meets requirements                                           │
│  - Failover works without service interruption                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Conclusion and Recommendations

### 8.1 Summary

The SIIT-DC model with Jool offers significant advantages for the CDN-POP project:

| Aspect | Improvement |
|--------|-------------|
| **Complexity** | ~50% reduction in configuration and code |
| **IPv4 Addresses** | Eliminates need for internal IPv4 infrastructure |
| **IPv6 Performance** | Native performance (no translation) |
| **IPv4 Performance** | Slight overhead (~0.5 µs latency) |
| **Operational** | Single protocol to manage and troubleshoot |
| **Future-proof** | IPv6-first architecture ready for IPv6-only Internet |

### 8.2 Recommendation

**RECOMMENDED: Adopt SIIT-DC with Jool**

The benefits of simplification significantly outweigh the minor overhead of IPv4 translation:

1. **Immediate Benefits:**
   - 50% reduction in infrastructure complexity
   - Elimination of IPv4 address scarcity for internal infrastructure
   - Simpler configuration management

2. **Long-term Benefits:**
   - Future-proof IPv6-first architecture
   - Easier scaling without IPv4 limitations
   - Reduced operational overhead

3. **Risk Mitigation:**
   - IPv4 clients continue to work transparently
   - Stateless translation has no connection tracking limits
   - Jool is production-ready and well-documented

### 8.3 Key Considerations

Before implementation, consider:

1. **Origin Server IPv6 Support:** Ensure origin servers support IPv6 connections
2. **Application Compatibility:** Verify applications don't have IPv4 hardcoded assumptions
3. **Monitoring:** Update monitoring systems for IPv6-only infrastructure
4. **Team Training:** Train team on Jool and IPv6 troubleshooting

### 8.4 Alternative: Hybrid Approach

If full SIIT-DC adoption is not immediately feasible, consider a **hybrid approach**:

1. **Phase 1:** Deploy Jool for new services (IPv6-only backend)
2. **Phase 2:** Migrate existing services gradually
3. **Phase 3:** Remove IPv4 from internal infrastructure

This allows incremental adoption while maintaining service continuity.

---

## Appendix A: Jool Quick Reference

### A.1 Common Commands

```bash
# Instance management
jool_siit instance add "name" --netfilter
jool_siit instance remove "name"
jool_siit --instance "name" enable
jool_siit --instance "name" disable

# EAM management
jool_siit --instance "name" eamt add 192.0.2.1 2001:db8::1
jool_siit --instance "name" eamt remove 192.0.2.1
jool_siit --instance "name" eamt display

# Global configuration
jool_siit --instance "name" update global pool6 "2001:db8::/96"
jool_siit --instance "name" display global

# Statistics
jool_siit --instance "name" stats display
jool_siit --instance "name" session display
```

### A.2 Troubleshooting

```bash
# Check translation
tcpdump -i eth0 -nn ip6
tcpdump -i eth0 -nn ip

# Jool status
jool_siit --instance "name" display global
jool_siit --instance "name" eamt display

# Kernel module
lsmod | grep jool
dmesg | grep jool

# Performance
cat /proc/net/jool_siit/name/stats
```

---

## Appendix B: RFC 6052 Address Calculator

```python
#!/usr/bin/env python3
# rfc6052.py - IPv4-embedded IPv6 address calculator

import ipaddress
import sys

def ipv4_to_ipv6(ipv4_str, prefix="2001:db8::/96"):
    """Convert IPv4 address to IPv6 using RFC 6052."""
    ipv4 = ipaddress.IPv4Address(ipv4_str)
    prefix_net = ipaddress.IPv6Network(prefix)
    
    # Get prefix bytes
    prefix_bytes = prefix_net.network_address.packed
    
    # Get IPv4 bytes
    ipv4_bytes = ipv4.packed
    
    # Calculate suffix length
    suffix_len = 16 - prefix_net.prefixlen // 8 - 4
    
    # Build IPv6 address
    ipv6_bytes = prefix_bytes[:prefix_net.prefixlen // 8]
    ipv6_bytes += ipv4_bytes
    ipv6_bytes += b'\x00' * suffix_len
    
    return ipaddress.IPv6Address(ipv6_bytes)

def ipv6_to_ipv4(ipv6_str, prefix="2001:db8::/96"):
    """Extract IPv4 address from IPv6 using RFC 6052."""
    ipv6 = ipaddress.IPv6Address(ipv6_str)
    prefix_net = ipaddress.IPv6Network(prefix)
    
    # Calculate IPv4 position
    ipv4_start = prefix_net.prefixlen // 8
    
    # Extract IPv4 bytes
    ipv6_bytes = ipv6.packed
    ipv4_bytes = ipv6_bytes[ipv4_start:ipv4_start + 4]
    
    return ipaddress.IPv4Address(ipv4_bytes)

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: rfc6052.py <ipv4-address|ipv6-address> [prefix]")
        sys.exit(1)
    
    addr = sys.argv[1]
    prefix = sys.argv[2] if len(sys.argv) > 2 else "2001:db8::/96"
    
    try:
        if '.' in addr:
            print(f"IPv4: {addr}")
            print(f"IPv6: {ipv4_to_ipv6(addr, prefix)}")
        else:
            print(f"IPv6: {addr}")
            print(f"IPv4: {ipv6_to_ipv4(addr, prefix)}")
    except Exception as e:
        print(f"Error: {e}")
        sys.exit(1)
```

---

## References

1. **RFC 6145** - IP/ICMP Translation Algorithm: https://datatracker.ietf.org/doc/html/rfc6145
2. **RFC 7756** - SIIT-DC: Stateless Translation: https://datatracker.ietf.org/doc/html/rfc7756
3. **RFC 6052** - IPv6 Addressing of IPv4/IPv6 Translators: https://datatracker.ietf.org/doc/html/rfc6052
4. **Jool Documentation**: https://jool.mx/en/
5. **Jool GitHub**: https://github.com/NICMx/Jool
6. **Katran Documentation**: https://github.com/facebookincubator/katran
7. **Fastly Fighting FIB**: https://www.fastly.com/blog/fighting-fib-scaling-routes-commodity-switches

---

*Document Version: 1.0*
*Last Updated: 2026-03-29*
*Author: CDN-POP Project Analysis*
