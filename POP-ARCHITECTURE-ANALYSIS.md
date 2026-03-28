# POP Architecture Analysis: Katran (Ingress) + Fastly Fighting FIB (Egress)

## Executive Summary

This document presents a comprehensive architecture for your Point of Presence (POP) that combines:

- **Ingress Traffic**: Meta's **Katran** - Layer 4 load balancer using XDP/eBPF for high-performance packet processing
- **Egress Traffic**: Fastly's **Fighting FIB** approach - Distributed routing using commodity switches with route reflection

This hybrid architecture leverages the strengths of both solutions while ensuring they operate seamlessly together without conflicts.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Step 1: Ingress with Katran](#2-step-1-ingress-with-katran)
3. [Step 2: Egress with Fastly Fighting FIB](#3-step-2-egress-with-fastly-fighting-fib)
4. [Integration: How Both Solutions Work Together](#4-integration-how-both-solutions-work-together)
5. [Implementation Checklist](#5-implementation-checklist)
6. [Network Diagrams](#6-network-diagrams)

---

## 1. Architecture Overview

### Design Principles

| Principle | Implementation |
|-----------|----------------|
| **Efficiency** | XDP processing before kernel for ingress; direct routing for egress |
| **Security** | DSR mode prevents LB bottleneck; distributed routing eliminates single points of failure |
| **Scalability** | Linear scaling with NIC queues (ingress); linear scaling with servers (egress) |
| **Cost-Effective** | Uses commodity switches (Arista) and standard Linux servers |

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                        │
│                    (Clients / Origin Servers)                                │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
    ┌─────────────────┐             ┌─────────────────┐
    │  Transit ISP 1  │             │  Transit ISP 2  │
    │    (eBGP)       │             │    (eBGP)       │
    └────────┬────────┘             └────────┬────────┘
             │                               │
             ▼                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         POP INFRASTRUCTURE                                   │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     BORDER SWITCHES (Arista)                           │  │
│  │  ┌─────────────┐    iBGP/ECMP    ┌─────────────┐                       │  │
│  │  │  Arista SW1 │◀──────────────▶│  Arista SW2 │                       │  │
│  │  │  (Primary)  │                 │ (Secondary)│                       │  │
│  │  └──────┬──────┘                 └──────┬──────┘                       │  │
│  └─────────┼───────────────────────────────┼──────────────────────────────┘  │
│            │                               │                                  │
│            │     ECMP (5-tuple hash)       │                                  │
│            └───────────────┬───────────────┘                                  │
│                            │                                                  │
│  ┌─────────────────────────▼─────────────────────────────────────────────┐   │
│  │                    KATRAN LOAD BALANCERS                               │   │
│  │  ┌──────────────────┐              ┌──────────────────┐                │   │
│  │  │   Katran LB-1    │              │   Katran LB-2    │                │   │
│  │  │   XDP/eBPF       │              │   XDP/eBPF       │                │   │
│  │  │   VIP Anycast    │              │   VIP Anycast    │                │   │
│  │  │   10.0.1.1       │              │   10.0.1.2       │                │   │
│  │  └────────┬─────────┘              └────────┬─────────┘                │   │
│  └───────────┼─────────────────────────────────┼──────────────────────────┘   │
│              │         IPIP Encap              │                              │
│              └────────────────┬────────────────┘                              │
│                               │                                               │
│  ┌────────────────────────────▼──────────────────────────────────────────┐   │
│  │                      EDGE SERVERS (6x)                                 │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐    │   │
│  │  │Edge-1  │ │Edge-2  │ │Edge-3  │ │Edge-4  │ │Edge-5  │ │Edge-6  │    │   │
│  │  │Cache   │ │Cache   │ │Cache   │ │Cache   │ │Cache   │ │Cache   │    │   │
│  │  │10.0.2.1│ │10.0.2.2│ │10.0.2.3│ │10.0.2.4│ │10.0.2.5│ │10.0.2.6│    │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └────────┘    │   │
│  │                                                                        │   │
│  │  Each server has:                                                      │   │
│  │  - VIP configured on loopback (for DSR)                               │   │
│  │  - IPIP interface for decapsulation                                    │   │
│  │  - BIRD instance for route reflection                                  │   │
│  └────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐   │
│  │                    MANAGEMENT / CONTROL PLANE                          │   │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐      │   │
│  │  │   Silverton      │  │   Healthcheck    │  │   Monitoring     │      │   │
│  │  │   Controller     │  │   Service        │  │   (Prometheus)   │      │   │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────┘      │   │
│  └────────────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Step 1: Ingress with Katran

### 2.1 Overview

Katran handles all **inbound traffic** from the internet to your POP. It operates at Layer 4, distributing TCP/UDP connections across your edge servers.

### 2.2 Katran Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    KATRAN LOAD BALANCER                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   XDP/eBPF Layer                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │   │
│  │  │  balancer   │  │  healthcheck│  │   root      │       │   │
│  │  │  .bpf.o     │  │  _ipip.o    │  │   xdp.o     │       │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   BPF Maps                               │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │   │
│  │  │ VIP Map  │ │ Real Map │ │ LRU Cache│ │ Stats Map│     │   │
│  │  │          │ │          │ │          │ │          │     │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Control Plane                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │   │
│  │  │ KatranLb    │  │  ExaBGP     │  │  gRPC/Thrift│       │   │
│  │  │  Library    │  │  (VIP adv)  │  │  API        │       │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Ingress Packet Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        INGRESS PACKET FLOW                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. CLIENT REQUEST                                                       │
│     ┌───────┐                                                            │
│     │Client │ ────── SYN ─────▶ VIP:203.0.113.10:443                     │
│     └───────┘                   (Anycast IP advertised via BGP)          │
│                                                                          │
│  2. SWITCH PROCESSING (ECMP)                                             │
│     ┌─────────┐                                                          │
│     │ Arista  │  Hash(5-tuple) → Select Katran LB                        │
│     │ Switch  │  src_ip + src_port + dst_ip + dst_port + proto           │
│     └────┬────┘                                                          │
│          │                                                               │
│          ▼                                                               │
│  3. KATRAN XDP PROCESSING (Before Kernel!)                               │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │  a) Check if dst_ip is configured VIP         ─────────┐     │     │
│     │  b) Check LRU cache for existing connection           │     │     │
│     │     - If found: use cached real server                │     │     │
│     │     - If not found: compute Maglev hash               │     │     │
│     │  c) Select real server based on hash ring             │     │     │
│     │  d) Update LRU cache (connection → real)               │     │     │
│     │  e) IPIP encapsulate packet                           ▼     │     │
│     └─────────────────────────────────────────────────────────────┐     │
│                              │                                        │     │
│                              ▼                                        │     │
│  4. IPIP ENCAPSULATION                                                  │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │  Original Packet:                                            │     │
│     │    src: Client_IP, dst: VIP                                  │     │
│     │                                                               │     │
│     │  Encapsulated Packet:                                         │     │
│     │    Outer: src: 172.16.X.Y (RSS-friendly), dst: Real_IP       │     │
│     │    Inner: src: Client_IP, dst: VIP (unchanged)              │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                              │                                          │
│                              ▼                                          │
│  5. FORWARD TO EDGE SERVER                                              │
│     ┌─────────┐                                                         │
│     │ Edge    │ ◀── IPIP packet (decapsulate)                          │
│     │ Server  │     - Remove outer header                              │
│     │         │     - Process inner packet (VIP on loopback)           │
│     └─────────┘                                                         │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Katran Configuration

#### KatranConfig Structure

```cpp
struct KatranConfig {
  std::string mainInterface;      // "eth0" - main interface for XDP
  std::string v4TunInterface;     // "ipip0" - IPv4 tunnel interface
  std::string v6TunInterface;     // "ipip60" - IPv6 tunnel interface
  std::string balancerProgPath;   // path to balancer.bpf.o
  std::string healthcheckingProgPath; // path to healthchecking_ipip.o
  std::vector<uint8_t> defaultMac;    // MAC of default gateway (switch)
  uint32_t priority = 10;             // TC filter priority
  std::string rootMapPath;            // For shared XDP mode
  uint32_t rootMapPos = 2;            // Position in prog array
  bool enableHc = true;               // Enable healthcheck forwarding
  uint32_t maxVips = 64;              // Maximum VIPs
  uint32_t maxReals = 128;            // Maximum real servers
  uint32_t chRingSize = 65537;        // Maglev ring size (prime!)
  uint64_t LruSize = 8000000;         // Connection tracking table size
  std::vector<int32_t> forwardingCores; // CPU cores for forwarding
  std::vector<int32_t> numaNodes;       // NUMA node hints
};
```

#### VIP and Real Configuration

```bash
# Add VIP (TCP port 443)
./katran_goclient -A -t 203.0.113.10:443

# Add real servers with weights
./katran_goclient -a -t 203.0.113.10:443 -r 10.0.2.1 -w 10
./katran_goclient -a -t 203.0.113.10:443 -r 10.0.2.2 -w 10
./katran_goclient -a -t 203.0.113.10:443 -r 10.0.2.3 -w 10
./katran_goclient -a -t 203.0.113.10:443 -r 10.0.2.4 -w 10
./katran_goclient -a -t 203.0.113.10:443 -r 10.0.2.5 -w 10
./katran_goclient -a -t 203.0.113.10:443 -r 10.0.2.6 -w 10

# List configuration
./katran_goclient -l
```

### 2.5 Edge Server Configuration (for Katran)

```bash
# Create IPIP interfaces for decapsulation
ip link add name ipip0 type ipip external
ip link add name ipip60 type ip6tnl external
ip link set up dev ipip0
ip link set up dev ipip60

# Configure dummy IP on IPIP interface
ip addr add 127.0.0.42/32 dev ipip0

# Configure VIP on loopback (for DSR)
ip addr add 203.0.113.10/32 dev lo

# Disable rp_filter (required for DSR)
for sc in $(sysctl -a | awk '/\.rp_filter/ {print $1}'); do
  sysctl ${sc}=0
done

# Increase MTU to accommodate IPIP overhead
ip link set dev eth0 mtu 9000
```

---

## 3. Step 2: Egress with Fastly Fighting FIB

### 3.1 Overview

For **outbound traffic** (responses from edge servers to clients, and traffic to origin servers), we implement Fastly's **Fighting FIB** approach. This eliminates the need for expensive routers by pushing routing intelligence to the switches and hosts.

### 3.2 Fighting FIB Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    FIGHTING FIB ARCHITECTURE                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PROBLEM: Switch FIB is limited (tens of thousands of routes)            │
│  SOLUTION: Push routes to hosts, use ARP manipulation                    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                    TRADITIONAL ROUTER APPROACH                      │ │
│  │                                                                     │ │
│  │  Internet ──▶ Router (600K+ routes in FIB) ──▶ Switch ──▶ Hosts    │ │
│  │                     (EXPENSIVE!)                                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                    FASTLY FIGHTING FIB APPROACH                     │ │
│  │                                                                     │ │
│  │  Internet ──▶ Switch (BGP daemon) ──iBGP──▶ Hosts (routes in kernel)│ │
│  │              (Commodity Arista)        │                            │ │
│  │                                       ▼                            │ │
│  │                              Host kernel FIB                        │ │
│  │                              (Full routing table)                   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Silverton - Route Reflection Controller

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    SILVERTON CONTROLLER                                   │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Silverton is a distributed routing agent that orchestrates:             │
│  1. BGP route reflection from switches to hosts                          │
│  2. ARP table propagation                                                │
│  3. Dynamic network configuration                                        │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                    ROUTE REFLECTION FLOW                             ││
│  │                                                                      ││
│  │  Transit ISP                                                         ││
│  │      │                                                               ││
│  │      │ eBGP (full internet routes)                                  ││
│  │      ▼                                                               ││
│  │  ┌──────────────┐                                                    ││
│  │  │   Arista     │  BIRD/ExaBGP daemon                               ││
│  │  │   Switch     │  (userspace BGP)                                  ││
│  │  │              │                                                    ││
│  │  │  Routes:     │                                                    ││
│  │  │  - Default   │                                                    ││
│  │  │  - Transit 1 │                                                    ││
│  │  │  - Transit 2 │                                                    ││
│  │  └──────┬───────┘                                                    ││
│  │         │                                                            ││
│  │         │ iBGP (route reflection)                                    ││
│  │         │                                                            ││
│  │         ▼                                                            ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               ││
│  │  │   Host 1     │  │   Host 2     │  │   Host N     │               ││
│  │  │   BIRD       │  │   BIRD       │  │   BIRD       │               ││
│  │  │   (kernel)   │  │   (kernel)   │  │   (kernel)   │               ││
│  │  │              │  │              │  │              │               ││
│  │  │  Full FIB    │  │  Full FIB    │  │  Full FIB    │               ││
│  │  │  in kernel   │  │  in kernel   │  │  in kernel   │               ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘               ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.4 ARP Propagation

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    ARP PROPAGATION MECHANISM                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Problem: Hosts know nexthop IPs but not MAC addresses                   │
│  Solution: Silverton propagates ARP entries from switch to hosts         │
│                                                                          │
│  Step 1: Switch learns provider MAC                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │  Arista Switch                                                       ││
│  │  Interface: Ethernet1                                                ││
│  │  Provider IP: 10.0.0.1                                               ││
│  │  Provider MAC: aa:bb:cc:dd:ee:ff                                     ││
│  │                                                                      ││
│  │  Silverton daemon subscribes to ARP table changes via EOS API       ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                              │                                           │
│                              │ Silverton propagates                      │
│                              ▼                                           │
│  Step 2: Hosts configured with provider reachability                     │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │  Edge Server (Host)                                                  ││
│  │                                                                      ││
│  │  # Make provider IP appear link-local                               ││
│  │  ip addr add 10.0.1.1 peer 10.0.0.1 dev eth0                        ││
│  │                                                                      ││
│  │  # Configure static ARP entry                                       ││
│  │  ip neigh replace 10.0.0.1 lladdr aa:bb:cc:dd:ee:ff nud perman     ││
│  │                                                                      ││
│  │  Now host can send directly to provider!                            ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  Step 3: Packet forwarding                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │  Host routing table:                                                 ││
│  │  default via 10.0.0.1 dev eth0                                      ││
│  │                                                                      ││
│  │  Host sends packet:                                                  ││
│  │  dst: Client_IP (from route lookup)                                 ││
│  │  nexthop: 10.0.0.1                                                   ││
│  │  MAC: aa:bb:cc:dd:ee:ff (from ARP table)                            ││
│  │                                                                      ││
│  │  Switch receives frame:                                              ││
│  │  - Looks up MAC in MAC address table                                ││
│  │  - Forwards to correct interface (toward provider)                  ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.5 Egress Packet Flow (DSR)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        EGRESS PACKET FLOW (DSR)                           │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. EDGE SERVER RESPONSE (Direct Server Return)                          │
│     ┌─────────┐                                                          │
│     │  Edge   │  Response packet:                                        │
│     │ Server  │    src: VIP (203.0.113.10)                               │
│     │         │    dst: Client_IP                                        │
│     │         │                                                          │
│     │         │  Route lookup:                                           │
│     │         │    - Check kernel FIB (full routes from iBGP)            │
│     │         │    - Select best path (transit 1 or 2)                   │
│     └────┬────┘                                                          │
│          │                                                               │
│          ▼                                                               │
│  2. DIRECT TO SWITCH (Bypasses Katran!)                                  │
│     ┌─────────┐                                                         │
│     │ Arista  │  Receives packet:                                       │
│     │ Switch  │    dst MAC: provider MAC (from host's ARP)              │
│     │         │                                                          │
│     │         │  Forwards based on:                                      │
│     │         │    - MAC address table lookup                           │
│     │         │    - Direct to transit provider                         │
│     └────┬────┘                                                          │
│          │                                                               │
│          ▼                                                               │
│  3. TO INTERNET                                                          │
│     ┌─────────┐                                                         │
│     │ Transit │  Packet goes directly to client                         │
│     │ ISP     │  Katran LB is completely bypassed!                      │
│     └─────────┘                                                         │
│                                                                          │
│  KEY INSIGHT:                                                            │
│  ─────────────                                                           │
│  Ingress: Client → Switch → Katran → Edge Server (IPIP encap)           │
│  Egress:  Edge Server → Switch → Transit → Client (Direct!)             │
│                                                                          │
│  This asymmetry is what makes DSR so efficient:                         │
│  - Load balancer only processes inbound traffic                         │
│  - Outbound traffic scales with number of edge servers                  │
│  - No bottleneck on response path                                       │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.6 BIRD Configuration for Route Reflection

#### Switch BIRD Configuration

```bash
# /etc/bird/bird.conf on Arista switch (via EOS API or container)

# eBGP session with transit providers
protocol bgp transit1 {
    local as 64512;
    neighbor 10.0.0.1 as 64513;
    import all;
    export all;
}

protocol bgp transit2 {
    local as 64512;
    neighbor 10.0.0.2 as 64514;
    import all;
    export all;
}

# iBGP route reflection to hosts
protocol bgp host_rr {
    local as 64512;
    neighbor 10.0.2.1 port 179 as 64512;  # Edge-1
    neighbor 10.0.2.2 port 179 as 64512;  # Edge-2
    neighbor 10.0.2.3 port 179 as 64512;  # Edge-3
    neighbor 10.0.2.4 port 179 as 64512;  # Edge-4
    neighbor 10.0.2.5 port 179 as 64512;  # Edge-5
    neighbor 10.0.2.6 port 179 as 64512;  # Edge-6
    
    import none;
    export all;
    rr client;
}
```

#### Host BIRD Configuration

```bash
# /etc/bird/bird.conf on each Edge Server

# iBGP session with switch (receive routes)
protocol bgp switch_rr {
    local as 64512;
    neighbor 10.0.1.254 as 64512;  # Switch IP
    import all;
    export none;
}

# Kernel route injection
protocol kernel {
    scan time 60;
    import none;
    export all;  # Export BGP routes to kernel
}

# Static route for VIP (advertised by Katran via ExaBGP)
protocol static {
    route 203.0.113.10/32 via "lo";
}
```

---

## 4. Integration: How Both Solutions Work Together

### 4.1 Key Integration Points

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    INTEGRATION ARCHITECTURE                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                    TRAFFIC DIRECTION SEPARATION                      ││
│  │                                                                      ││
│  │  INGRESS (Inbound)              EGRESS (Outbound)                   ││
│  │  ──────────────────             ──────────────────                  ││
│  │                                                                      ││
│  │  Technology: Katran            Technology: Fighting FIB             ││
│  │  Layer: L4 Load Balancing      Layer: L3 Routing                    ││
│  │  Mechanism: XDP + IPIP         Mechanism: BGP + ARP                 ││
│  │                                                                      ││
│  │  ┌─────────────────┐           ┌─────────────────┐                  ││
│  │  │                 │           │                 │                  ││
│  │  │   ┌─────────┐   │           │   ┌─────────┐   │                  ││
│  │  │   │ Client  │───┼──────────▶│   │ Switch  │◀──┼─── Edge Server  ││
│  │  │   └─────────┘   │           │   └─────────┘   │                  ││
│  │  │                 │           │        │        │                  ││
│  │  │   ┌─────────┐   │           │        ▼        │                  ││
│  │  │   │ Switch  │   │           │   ┌─────────┐   │                  ││
│  │  │   └─────────┘   │           │   │ Transit │   │                  ││
│  │  │        │        │           │   └─────────┘   │                  ││
│  │  │        ▼        │           │        │        │                  ││
│  │  │   ┌─────────┐   │           │        ▼        │                  ││
│  │  │   │ Katran  │   │           │   ┌─────────┐   │                  ││
│  │  │   └─────────┘   │           │   │ Client  │   │                  ││
│  │  │        │        │           │   └─────────┘   │                  ││
│  │  │        ▼        │           │                 │                  ││
│  │  │   ┌─────────┐   │           │                 │                  ││
│  │  │   │  Edge   │───┼───────────┼─────────────────┘                  ││
│  │  │   └─────────┘   │           │                 │                  ││
│  │  │                 │           │                 │                  ││
│  │  └─────────────────┘           └─────────────────┘                  ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Why No Mismatch Occurs

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    NO MISMATCH EXPLANATION                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  CONCERN: Could ingress and egress paths conflict?                       │
│  ANSWER: No, they operate at different layers and directions             │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ 1. DIFFERENT TRAFFIC DIRECTIONS                                      ││
│  │                                                                      ││
│  │    INGRESS: Client → POP (request traffic)                          ││
│  │    EGRESS:  POP → Client (response traffic)                         ││
│  │             POP → Origin (fetch traffic)                            ││
│  │                                                                      ││
│  │    These are completely separate flows!                             ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ 2. DIFFERENT PROCESSING LAYERS                                       ││
│  │                                                                      ││
│  │    Katran (Ingress):                                                 ││
│  │    - Operates at Layer 4 (TCP/UDP)                                  ││
│  │    - Matches on VIP + Port                                          ││
│  │    - Modifies packet: IPIP encapsulation                            ││
│  │    - Decision: Which edge server to use                             ││
│  │                                                                      ││
│  │    Fighting FIB (Egress):                                            ││
│  │    - Operates at Layer 3 (IP routing)                               ││
│  │    - Matches on destination IP (client or origin)                   ││
│  │    - No packet modification (except MAC for next hop)               ││
│  │    - Decision: Which transit provider to use                        ││
│  │                                                                      ││
│  │    Different decision domains = No conflict!                        ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ 3. DSR (Direct Server Return) ENSURES SEPARATION                     ││
│  │                                                                      ││
│  │    Ingress path:                                                     ││
│  │    Client → Switch → Katran → IPIP → Edge Server                    ││
│  │                                                                      ││
│  │    Egress path:                                                      ││
│  │    Edge Server → Switch → Transit → Client                          ││
│  │                                                                      ││
│  │    Response traffic NEVER goes through Katran!                      ││
│  │    This is the key architectural decision that prevents conflicts.  ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ 4. VIP CONFIGURATION IS CONSISTENT                                   ││
│  │                                                                      ││
│  │    Katran:                                                           ││
│  │    - VIP 203.0.113.10 configured as service to balance              ││
│  │    - Advertised via BGP to internet (anycast)                       ││
│  │                                                                      ││
│  │    Edge Servers:                                                     ││
│  │    - VIP 203.0.113.10 configured on loopback                        ││
│  │    - Used as source IP for responses (DSR)                          ││
│  │                                                                      ││
│  │    Both use the SAME VIP, but for different purposes:               ││
│  │    - Katran: receives requests TO the VIP                           ││
│  │    - Edge: sends responses FROM the VIP                             ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Complete Bidirectional Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE BIDIRECTIONAL FLOW                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  TIME                                                                    │
│   │                                                                      │
│   │  ┌──────────────────────────────────────────────────────────────┐  │
│   │  │ T1: CLIENT SYN REQUEST                                        │  │
│   │  │                                                                │  │
│   │  │  Client (203.0.113.100:54321)                                 │  │
│   │  │       │                                                        │  │
│   │  │       │ IP: src=203.0.113.100, dst=203.0.113.10 (VIP)         │  │
│   │  │       │ TCP: SYN, dst_port=443                                │  │
│   │  │       ▼                                                        │  │
│   │  │  Internet Routing (Anycast)                                   │  │
│   │  │       │                                                        │  │
│   │  │       ▼                                                        │  │
│   │  │  Transit ISP                                                   │  │
│   │  │       │                                                        │  │
│   │  │       ▼                                                        │  │
│   │  │  Arista Switch (ECMP hash on 5-tuple)                         │  │
│   │  │       │                                                        │  │
│   │  │       │ Hash(203.0.113.100, 54321, 203.0.113.10, 443, TCP)    │  │
│   │  │       │ → Select Katran LB-1                                  │  │
│   │  │       ▼                                                        │  │
│   │  │  Katran LB-1 (XDP processing)                                 │  │
│   │  │       │                                                        │  │
│   │  │       │ 1. VIP match: 203.0.113.10:443 ✓                      │  │
│   │  │       │ 2. LRU miss (new connection)                          │  │
│   │  │       │ 3. Maglev hash → Select Edge-3 (10.0.2.3)             │  │
│   │  │       │ 4. IPIP encapsulate                                   │  │
│   │  │       ▼                                                        │  │
│   │  │  IPIP Packet:                                                  │  │
│   │  │       Outer: src=172.16.0.3, dst=10.0.2.3                     │  │
│   │  │       Inner: src=203.0.113.100, dst=203.0.113.10              │  │
│   │  │       │                                                        │  │
│   │  │       ▼                                                        │  │
│   │  │  Edge Server 3 (10.0.2.3)                                     │  │
│   │  │       │                                                        │  │
│   │  │       │ 1. Decapsulate IPIP                                   │  │
│   │  │       │ 2. VIP on loopback: accept packet                     │  │
│   │  │       │ 3. TCP stack: SYN received, prepare SYN-ACK           │  │
│   │  └──────────────────────────────────────────────────────────────┘  │
│   │                                                                      │
│   │  ┌──────────────────────────────────────────────────────────────┐  │
│   │  │ T2: EDGE SERVER SYN-ACK RESPONSE (DSR - Direct Server Return) │  │
│   │  │                                                                │  │
│   │  │  Edge Server 3                                                 │  │
│   │  │       │                                                        │  │
│   │  │       │ IP: src=203.0.113.10 (VIP), dst=203.0.113.100         │  │
│   │  │       │ TCP: SYN-ACK, src_port=443                            │  │
│   │  │       │                                                        │  │
│   │  │       │ Kernel FIB lookup:                                    │  │
│   │  │       │   dst 203.0.113.100 → via Transit 1 (best path)       │  │
│   │  │       │   nexthop: 10.0.0.1 (Transit 1 gateway)               │  │
│   │  │       │                                                        │  │
│   │  │       │ ARP lookup:                                           │  │
│   │  │       │   10.0.0.1 → MAC aa:bb:cc:dd:ee:ff                   │  │
│   │  │       ▼                                                        │  │
│   │  │  Ethernet Frame:                                               │  │
│   │  │       dst MAC: aa:bb:cc:dd:ee:ff (Transit 1)                  │  │
│   │  │       src MAC: 11:22:33:44:55:66 (Edge-3 NIC)                 │  │
│   │  │       │                                                        │  │
│   │  │       ▼                                                        │  │
│   │  │  Arista Switch                                                  │  │
│   │  │       │                                                        │  │
│   │  │       │ MAC table lookup:                                      │  │
│   │  │       │   aa:bb:cc:dd:ee:ff → Ethernet1 (Transit 1)           │  │
│   │  │       │                                                        │  │
│   │  │       │ Forward directly to Transit 1                         │  │
│   │  │       │ (Katran is COMPLETELY BYPASSED!)                       │  │
│   │  │       ▼                                                        │  │
│   │  │  Transit ISP 1                                                  │  │
│   │  │       │                                                        │  │
│   │  │       ▼                                                        │  │
│   │  │  Internet Routing                                               │  │
│   │  │       │                                                        │  │
│   │  │       ▼                                                        │  │
│   │  │  Client (203.0.113.100:54321)                                 │  │
│   │  │       │                                                        │  │
│   │  │       └── Received SYN-ACK from VIP!                          │  │
│   │  └──────────────────────────────────────────────────────────────┘  │
│   │                                                                      │
│   ▼                                                                      │
│  TIME                                                                    │
│                                                                          │
│  KEY OBSERVATIONS:                                                       │
│  ─────────────────                                                       │
│  1. Ingress path: Client → Switch → Katran → Edge (IPIP)               │
│  2. Egress path: Edge → Switch → Transit → Client (Direct)             │
│  3. Katran only sees request traffic (inbound)                          │
│  4. Response traffic bypasses Katran entirely (DSR)                     │
│  5. Fighting FIB handles egress routing (BGP routes in host kernel)     │
│  6. No mismatch: different paths, different layers, different purpose  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.4 Configuration Summary Table

| Component | Configuration | Purpose |
|-----------|---------------|---------|
| **Katran LB** | VIP: 203.0.113.10:443 | Receive inbound requests |
| **Katran LB** | Reals: 10.0.2.1-6 | Forward to edge servers |
| **Katran LB** | ExaBGP | Advertise VIP to switches |
| **Edge Server** | VIP on loopback | Accept packets for VIP (DSR) |
| **Edge Server** | IPIP interface | Decapsulate incoming packets |
| **Edge Server** | BIRD (iBGP client) | Receive full routes from switch |
| **Edge Server** | Static ARP | Direct forwarding to transit |
| **Arista Switch** | BIRD (route reflector) | Reflect routes to hosts |
| **Arista Switch** | ECMP | Distribute to Katran LBs |
| **Arista Switch** | Silverton | Propagate ARP entries |

---

## 5. Implementation Checklist

### Phase 1: Infrastructure Setup

- [ ] **Switch Configuration**
  - [ ] Configure Arista switches with basic L3 routing
  - [ ] Establish eBGP sessions with transit providers
  - [ ] Configure iBGP between switches (route reflection)
  - [ ] Enable ECMP for load balancer distribution
  - [ ] Install and configure BIRD on switches

- [ ] **Network Topology**
  - [ ] Configure subnets:
    - [ ] 10.0.1.0/24 - Load Balancers
    - [ ] 10.0.2.0/24 - Edge Servers
  - [ ] Configure MTU 9000 (jumbo frames for IPIP)
  - [ ] Verify L3 routing between all components

### Phase 2: Katran Load Balancers

- [ ] **Server Provisioning**
  - [ ] Provision 2 servers for Katran (32+ cores, 64GB RAM recommended)
  - [ ] Install Linux kernel 5.6+
  - [ ] Enable BPF JIT: `sysctl net.core.bpf_jit_enable=1`

- [ ] **Katran Installation**
  - [ ] Clone Katran repository
  - [ ] Install dependencies (folly, glog, gflags, etc.)
  - [ ] Build Katran: `./build_katran.sh`
  - [ ] Build BPF programs

- [ ] **Katran Configuration**
  - [ ] Create IPIP interfaces: `ipip0`, `ipip60`
  - [ ] Configure IRQ affinity for RSS optimization
  - [ ] Disable LRO/GRO on main interface
  - [ ] Configure default MAC (switch gateway)
  - [ ] Start Katran server (gRPC or Thrift)

- [ ] **VIP Configuration**
  - [ ] Add VIPs via gRPC client
  - [ ] Add real servers with weights
  - [ ] Configure healthcheck endpoints
  - [ ] Verify with `katran_goclient -l`

- [ ] **BGP Configuration**
  - [ ] Install ExaBGP
  - [ ] Configure VIP advertisement
  - [ ] Verify route propagation to switches

### Phase 3: Edge Servers

- [ ] **Server Configuration (repeat for all 6)**
  - [ ] Install Linux with kernel 5.6+
  - [ ] Create IPIP interfaces for decapsulation
  - [ ] Configure VIP on loopback interface
  - [ ] Disable rp_filter
  - [ ] Configure MTU 9000

- [ ] **BIRD Installation**
  - [ ] Install BIRD routing daemon
  - [ ] Configure iBGP session with switch
  - [ ] Configure kernel protocol (route injection)
  - [ ] Verify route reception: `birdc show route`

- [ ] **Silverton Agent**
  - [ ] Install Silverton client agent
  - [ ] Configure to receive ARP updates
  - [ ] Verify ARP propagation

- [ ] **Cache/CDN Software**
  - [ ] Install Varnish/NGINX/ATS
  - [ ] Configure to bind to VIP address
  - [ ] Test local functionality

### Phase 4: Integration Testing

- [ ] **Ingress Testing**
  - [ ] Send test request to VIP from external client
  - [ ] Verify ECMP distribution to Katran LBs
  - [ ] Verify IPIP encapsulation (tcpdump)
  - [ ] Verify packet arrival at correct edge server
  - [ ] Check Katran stats: `katran_goclient -s -lru`

- [ ] **Egress Testing**
  - [ ] Verify response goes directly to switch
  - [ ] Verify response bypasses Katran (DSR)
  - [ ] Verify correct transit selection
  - [ ] Check client receives response

- [ ] **Failover Testing**
  - [ ] Stop one Katran LB, verify traffic shifts
  - [ ] Stop one edge server, verify connections redistribute
  - [ ] Verify graceful draining (weight=0)

### Phase 5: Monitoring & Operations

- [ ] **Monitoring Setup**
  - [ ] Configure Prometheus metrics export
  - [ ] Create Grafana dashboards
  - [ ] Set up alerts for:
    - [ ] LB health
    - [ ] Edge server health
    - [ ] Connection rates
    - [ ] Error rates

- [ ] **Operational Runbooks**
  - [ ] Document server draining procedure
  - [ ] Document VIP addition/removal
  - [ ] Document troubleshooting steps
  - [ ] Create incident response procedures

---

## 6. Network Diagrams

### 6.1 Physical Topology

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PHYSICAL TOPOLOGY                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│     Transit 1                        Transit 2                              │
│     (10Gbps)                         (10Gbps)                               │
│         │                                │                                  │
│    ┌────┴────┐                      ┌────┴────┐                            │
│    │  Port 1 │                      │  Port 1 │                            │
│    │  Port 2 │                      │  Port 2 │                            │
│    └────┬────┘                      └────┬────┘                            │
│         │                                │                                  │
│  ┌──────┴────────────────────────────────┴──────┐                         │
│  │                                               │                         │
│  │  ┌─────────────────────────────────────────┐ │                         │
│  │  │           Arista Switch 1               │ │                         │
│  │  │         (7050X3 or similar)             │ │                         │
│  │  │                                         │ │                         │
│  │  │  Ports:                                 │ │                         │
│  │  │  - 2x 10G (transit uplinks)            │ │                         │
│  │  │  - 8x 10G (downlinks to servers)       │ │                         │
│  │  │  - 2x 40G (inter-switch)               │ │                         │
│  │  └─────────────────────────────────────────┘ │                         │
│  │                                               │                         │
│  │  ┌─────────────────────────────────────────┐ │                         │
│  │  │           Arista Switch 2               │ │                         │
│  │  │         (7050X3 or similar)             │ │                         │
│  │  │                                         │ │                         │
│  │  │  Same port configuration                │ │                         │
│  │  └─────────────────────────────────────────┘ │                         │
│  │                                               │                         │
│  └───────────────────────────────────────────────┘                         │
│         │                                │                                  │
│         │  Inter-switch link (40G)       │                                  │
│         └────────────────────────────────┘                                  │
│                                                                             │
│  Server Rack:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  ┌──────────┐  ┌──────────┐                                        │   │
│  │  │ Katran   │  │ Katran   │  Load Balancers (2x)                   │   │
│  │  │ LB-1     │  │ LB-2     │  - 32+ cores                          │   │
│  │  │          │  │          │  - 64GB RAM                           │   │
│  │  │ 10G NIC  │  │ 10G NIC  │  - XDP capable NIC                    │   │
│  │  └──────────┘  └──────────┘                                        │   │
│  │                                                                     │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                           │   │
│  │  │ Edge-1   │ │ Edge-2   │ │ Edge-3   │  Edge Servers (6x)        │   │
│  │  │ Cache    │ │ Cache    │ │ Cache    │  - 16+ cores             │   │
│  │  │ 10G NIC  │ │ 10G NIC  │ │ 10G NIC  │  - 128GB RAM             │   │
│  │  └──────────┘ └──────────┘ └──────────┘  - SSD cache storage     │   │
│  │                                                                     │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                           │   │
│  │  │ Edge-4   │ │ Edge-5   │ │ Edge-6   │                           │   │
│  │  │ Cache    │ │ Cache    │ │ Cache    │                           │   │
│  │  │ 10G NIC  │ │ 10G NIC  │ │ 10G NIC  │                           │   │
│  │  └──────────┘ └──────────┘ └──────────┘                           │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Logical Topology

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          LOGICAL TOPOLOGY                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                              Internet                                       │
│                                 │                                           │
│                    ┌────────────┴────────────┐                              │
│                    │                         │                              │
│               Transit 1                 Transit 2                           │
│              (AS 64513)               (AS 64514)                            │
│                    │                         │                              │
│                    │ eBGP                    │ eBGP                         │
│                    ▼                         ▼                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     AS 64512 (Your POP)                              │   │
│  │                                                                      │   │
│  │  ┌───────────────────────────────────────────────────────────────┐  │   │
│  │  │                    Routing Layer                               │  │   │
│  │  │                                                                │  │   │
│  │  │   ┌─────────────┐         ┌─────────────┐                     │  │   │
│  │  │   │  Arista SW1 │◀─iBGP──▶│  Arista SW2 │                     │  │   │
│  │  │   │             │         │             │                     │  │   │
│  │  │   │ BGP RR      │         │ BGP RR      │                     │  │   │
│  │  │   │ Silverton   │         │ Silverton   │                     │  │   │
│  │  │   └──────┬──────┘         └──────┬──────┘                     │  │   │
│  │  └──────────┼────────────────────────┼───────────────────────────┘  │   │
│  │             │                        │                               │   │
│  │             │    ECMP (VIP routes)   │                               │   │
│  │             └───────────┬────────────┘                               │   │
│  │                         │                                            │   │
│  │  ┌──────────────────────▼────────────────────────────────────────┐  │   │
│  │  │                 Load Balancing Layer                           │  │   │
│  │  │                                                                │  │   │
│  │  │   ┌─────────────────┐       ┌─────────────────┐               │  │   │
│  │  │   │    Katran LB-1  │       │    Katran LB-2  │               │  │   │
│  │  │   │                 │       │                 │               │  │   │
│  │  │   │ VIP: 203.0.113.10       │ VIP: 203.0.113.10              │  │   │
│  │  │   │ XDP/eBPF        │       │ XDP/eBPF        │               │  │   │
│  │  │   │ ExaBGP          │       │ ExaBGP          │               │  │   │
│  │  │   │ 10.0.1.1        │       │ 10.0.1.2        │               │  │   │
│  │  │   └────────┬────────┘       └────────┬────────┘               │  │   │
│  │  └────────────┼─────────────────────────┼────────────────────────┘  │   │
│  │               │      IPIP Encap         │                            │   │
│  │               └───────────┬─────────────┘                            │   │
│  │                           │                                          │   │
│  │  ┌────────────────────────▼────────────────────────────────────────┐  │   │
│  │  │                    Edge Server Layer                            │  │   │
│  │  │                                                                 │  │   │
│  │  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌─────┐ │  │   │
│  │  │  │Edge-1  │ │Edge-2  │ │Edge-3  │ │Edge-4  │ │Edge-5  │ │Edge-6│ │  │   │
│  │  │  │10.0.2.1│ │10.0.2.2│ │10.0.2.3│ │10.0.2.4│ │10.0.2.5│ │10.0.2.6│ │  │   │
│  │  │  │        │ │        │ │        │ │        │ │        │ │       │ │  │   │
│  │  │  │BIRD    │ │BIRD    │ │BIRD    │ │BIRD    │ │BIRD    │ │BIRD   │ │  │   │
│  │  │  │iBGP    │ │iBGP    │ │iBGP    │ │iBGP    │ │iBGP    │ │iBGP   │ │  │   │
│  │  │  │Cache   │ │Cache   │ │Cache   │ │Cache   │ │Cache   │ │Cache  │ │  │   │
│  │  │  │VIP/lo  │ │VIP/lo  │ │VIP/lo  │ │VIP/lo  │ │VIP/lo  │ │VIP/lo │ │  │   │
│  │  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └───────┘ │  │   │
│  │  │                                                                 │  │   │
│  │  │  Each server:                                                   │  │   │
│  │  │  - Full routing table (from iBGP)                              │  │   │
│  │  │  - VIP on loopback (for DSR)                                   │  │   │
│  │  │  - Static ARP (from Silverton)                                 │  │   │
│  │  └─────────────────────────────────────────────────────────────────┘  │   │
│  │                                                                       │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Traffic Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        TRAFFIC FLOW DIAGRAM                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  INGRESS (Request)                     EGRESS (Response)                    │
│  ══════════════════                    ══════════════════                    │
│                                                                             │
│  Client                                 Edge Server                         │
│    │                                        │                               │
│    │ 1. Request to VIP                      │                               │
│    │    (Anycast routing)                   │                               │
│    ▼                                        │                               │
│  Internet                               6. Response from VIP               │
│    │                                    (Direct to switch)                 │
│    │ 2. Route to POP                        │                               │
│    ▼                                        ▼                               │
│  Transit ISP ◀───────────────────────── Arista Switch                      │
│    │                                        ▲                               │
│    │ 3. Deliver to POP                      │ 7. MAC lookup                 │
│    ▼                                        │    (direct to transit)        │
│  Arista Switch                              │                               │
│    │                                        │                               │
│    │ 4. ECMP hash                           │                               │
│    │    (5-tuple)                           │                               │
│    ▼                                        │                               │
│  Katran LB ◀────────────────────────────────┘                               │
│    │                                                                         │
│    │ 5. IPIP encap                                                           │
│    │    (select edge server)                                                 │
│    ▼                                                                         │
│  Edge Server ◀───────────────────────────────────────────────────────────── │
│    │                                                                         │
│    └─────────────────────────────────────────────────────────────────────────│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         PACKET STRUCTURES                            │   │
│  │                                                                      │   │
│  │  INGRESS (after Katran):                                             │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ Ethernet │ Outer IP │ Inner IP │ TCP │ Data                   │  │   │
│  │  │          │ src: 172.16.X.Y     │ src: Client                 │  │   │
│  │  │ dst: Edge│ dst: Edge IP       │ dst: VIP                     │  │   │
│  │  │          │ proto: IPIP (4)    │ ports: Client:443            │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  │                                                                      │   │
│  │  EGRESS (from Edge Server):                                          │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ Ethernet │ IP │ TCP │ Data                                     │  │   │
│  │  │ dst: Transit│ src: VIP                                        │  │   │
│  │  │ src: Edge   │ dst: Client                                     │  │   │
│  │  │             │ ports: 443:Client                               │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  │                                                                      │   │
│  │  Note: Egress has NO IPIP encapsulation (DSR)                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

This architecture successfully combines:

1. **Katran (Ingress)**: High-performance L4 load balancing using XDP/eBPF
   - Handles all incoming client requests
   - IPIP encapsulation for forwarding to edge servers
   - Maglev consistent hashing for distribution
   - DSR mode for efficient response handling

2. **Fighting FIB (Egress)**: Distributed routing using commodity switches
   - Route reflection from switches to hosts
   - Full routing table in host kernels
   - ARP propagation via Silverton
   - Direct forwarding to transit providers

**Key Integration Points:**
- VIP is consistent across Katran and edge servers
- DSR ensures response traffic bypasses load balancers
- Different processing layers (L4 vs L3) prevent conflicts
- Separate traffic directions (inbound vs outbound) ensure no mismatch

This hybrid approach provides:
- ✅ **Efficiency**: XDP processing + direct routing
- ✅ **Security**: No single point of failure, distributed architecture
- ✅ **Scalability**: Linear scaling in both directions
- ✅ **Cost-effectiveness**: Uses commodity hardware throughout
