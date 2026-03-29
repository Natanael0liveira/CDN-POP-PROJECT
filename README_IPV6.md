# IPv6 Adaptation Guide for CDN-POP Architecture

## Executive Summary

This document details the structural changes and adaptations required to enable full IPv6 support in the CDN-POP architecture, focusing on **Katran** (load balancer) and **Silverton** (route reflection controller). The proposed solutions follow industry best practices and are designed to maintain high performance while supporting dual-stack (IPv4/IPv6) operation.

---

## Table of Contents

1. [Current Architecture Assessment](#1-current-architecture-assessment)
2. [IPv6 Best Practices Compliance](#2-ipv6-best-practices-compliance)
3. [Performance Considerations](#3-performance-considerations)
4. [Katran IPv6 Adaptation](#4-katran-ipv6-adaptation)
5. [Silverton IPv6 Adaptation](#5-silverton-ipv6-adaptation)
6. [BIRD Configuration for IPv6](#6-bird-configuration-for-ipv6)
7. [Traffic Steering IPv6 Support](#7-traffic-steering-ipv6-support)
8. [Implementation Checklist](#8-implementation-checklist)
9. [Why These Solutions Were Chosen](#9-why-these-solutions-were-chosen)

---

## 1. Current Architecture Assessment

### 1.1 IPv4-Centric Design

The current architecture is primarily designed for IPv4:

| Component | Current State | IPv6 Readiness |
|-----------|---------------|----------------|
| **Katran VIPs** | IPv4 addresses (203.0.113.x) | Partial (ipip60 mentioned) |
| **IPIP Tunnels** | IPv4-in-IPv4 only | Interface defined, mode unclear |
| **Silverton** | ARP propagation only | No NDP support |
| **BIRD** | IPv4 eBGP/iBGP | No IPv6 sessions |
| **Traffic Steering** | IPv4 IPSets | IPv6 sets mentioned but incomplete |

### 1.2 What Works Well

The architecture already demonstrates good foundational design:

1. **Dual Interface Definition**: `v6TunInterface` in KatranConfig shows awareness of IPv6
2. **Protocol-Agnostic Design**: DSR (Direct Server Return) works for both IPv4 and IPv6
3. **Modular Components**: Each component can be updated independently
4. **nftables Support**: Traffic steering rules already include `ip6 daddr` matching

---

## 2. IPv6 Best Practices Compliance

### 2.1 Best Practices Followed

| Best Practice | Implementation | Status |
|---------------|----------------|--------|
| **Dual-Stack Operation** | Run IPv4 and IPv6 simultaneously | ✅ Proposed |
| **No Tunneling Where Possible** | Direct routing for egress (DSR) | ✅ Already implemented |
| **Proper Address Planning** | /64 subnets for IPv6 | ✅ Proposed |
| **NDP Instead of ARP** | Silverton NDP propagation | ✅ Proposed |
| **Separate BGP Sessions** | IPv4 and IPv6 eBGP/iBGP | ✅ Proposed |
| **MTU Considerations** | 1280 byte minimum for IPv6 | ✅ Proposed |
| **Extension Header Handling** | Minimal encapsulation overhead | ✅ Proposed |

### 2.2 RFC Compliance

| RFC | Topic | Compliance |
|-----|-------|------------|
| **RFC 6434** | IPv6 Node Requirements | ✅ Dual-stack operation |
| **RFC 7706** | BGP IPv6 Operations | ✅ Separate IPv6 sessions |
| **RFC 4861** | NDP Specification | ✅ Silverton NDP support |
| **RFC 2473** | IPv6-in-IPv6 Tunneling | ✅ ip6ip6 mode |
| **RFC 6437** | IPv6 Flow Label | ⚠️ Optional enhancement |

### 2.3 Areas Needing Improvement

1. **Flow Label Usage**: IPv6 flow labels could enhance hash distribution
2. **Extension Headers**: Need to handle IPv6 extension headers in XDP
3. **Path MTU Discovery**: IPv6 requires PMTUD (no fragmentation)

---

## 3. Performance Considerations

### 3.1 IPv6 vs IPv4 Performance Impact

| Aspect | IPv4 | IPv6 | Impact |
|--------|------|------|--------|
| **Header Size** | 20 bytes | 40 bytes | +20 bytes overhead |
| **Address Lookup** | 32-bit | 128-bit | ~3x hash computation |
| **Checksum** | Required | Not required | Slight CPU reduction |
| **Encapsulation** | IPIP (8 bytes) | ip6ip6 (40 bytes) | +32 bytes overhead |

### 3.2 XDP Performance with IPv6

```
┌─────────────────────────────────────────────────────────────────┐
│                    XDP PROCESSING COMPARISON                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  IPv4 Packet Processing:                                         │
│  ┌─────────┬──────────┬──────────┬─────┬─────────┐              │
│  │ Eth 14B │ IPv4 20B │ TCP 20B  │ ... │ Total   │              │
│  └─────────┴──────────┴──────────┴─────┴─────────┘              │
│            └─ Parse: ~50 instructions ─┘                        │
│                                                                  │
│  IPv6 Packet Processing:                                         │
│  ┌─────────┬──────────┬──────────┬─────┬─────────┐              │
│  │ Eth 14B │ IPv6 40B │ TCP 20B  │ ... │ Total   │              │
│  └─────────┴──────────┴──────────┴─────┴─────────┘              │
│            └─ Parse: ~70 instructions ─┘                        │
│                                                                  │
│  Performance Impact:                                             │
│  - Header parsing: ~40% more instructions                        │
│  - Hash computation: ~3x more data to hash                      │
│  - Overall: ~10-15% reduction in PPS                            │
│                                                                  │
│  Mitigation:                                                     │
│  - Use IPv6 flow label in hash (RFC 6437)                       │
│  - Optimize BPF map key sizes                                   │
│  - Leverage NIC hardware offloads                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Hash Performance Optimization

**Why this matters**: Maglev hashing must process 5-tuple for every packet.

```c
// IPv4 Hash Input (96 bits total)
struct ipv4_5tuple {
    u32 src_ip;     // 32 bits
    u32 dst_ip;     // 32 bits
    u16 src_port;   // 16 bits
    u16 dst_port;   // 16 bits
    u8 proto;       // 8 bits
};  // Total: 104 bits (13 bytes)

// IPv6 Hash Input (288 bits total)
struct ipv6_5tuple {
    struct in6_addr src_ip;  // 128 bits
    struct in6_addr dst_ip;  // 128 bits
    u16 src_port;            // 16 bits
    u16 dst_port;            // 16 bits
    u8 proto;                // 8 bits
};  // Total: 296 bits (37 bytes)
```

**Optimization Strategy**: Use IPv6 Flow Label (20 bits) as pre-computed hash:

```c
// Optimized IPv6 hash using flow label
static __always_inline u32 hash_ipv6_5tuple(struct ipv6hdr *ip6, struct tcphdr *tcp) {
    u32 hash = 0;
    
    // Use flow label as base hash (RFC 6437)
    hash = ip6->flow_lbl[0] << 16 | ip6->flow_lbl[1] << 8 | ip6->flow_lbl[2];
    
    // Mix in ports
    hash ^= tcp->source ^ tcp->dest;
    
    // If flow label not set, fall back to full hash
    if (hash == 0) {
        hash = jhash(&ip6->saddr, 16, 0);
        hash = jhash(&ip6->daddr, 16, hash);
        hash ^= tcp->source ^ tcp->dest;
    }
    
    return hash;
}
```

### 3.4 Memory Impact

| Data Structure | IPv4 Size | IPv6 Size | Increase |
|----------------|-----------|-----------|----------|
| **VIP Map Key** | 4 bytes | 16 bytes | 4x |
| **Real Server Entry** | 4 bytes | 16 bytes | 4x |
| **LRU Cache Entry** | 12 bytes | 36 bytes | 3x |
| **Stats Map Key** | 4 bytes | 16 bytes | 4x |

**Memory Planning**: For 1M concurrent connections:
- IPv4: ~12 MB LRU cache
- IPv6: ~36 MB LRU cache

---

## 4. Katran IPv6 Adaptation

### 4.1 Why Katran Needs Changes

Katran operates at Layer 4 using XDP/eBPF, processing packets before the kernel. The current implementation:

1. **Parses IPv4 headers only** - `struct iphdr` parsing
2. **Uses 32-bit VIP keys** - BPF maps indexed by IPv4 address
3. **IPIP encapsulation** - IPv4-in-IPv4 tunneling

### 4.2 Proposed Solution: Dual-Stack XDP

```
┌─────────────────────────────────────────────────────────────────┐
│                    DUAL-STACK XDP ARCHITECTURE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────────┐                           │
│                    │   XDP Entry     │                           │
│                    └────────┬────────┘                           │
│                             │                                    │
│                    ┌────────▼────────┐                           │
│                    │ Ethertype Check │                           │
│                    └────────┬────────┘                           │
│                             │                                    │
│              ┌──────────────┼──────────────┐                     │
│              │              │              │                     │
│     ┌────────▼───┐  ┌──────▼──────┐  ┌────▼─────┐                │
│     │ 0x0800     │  │ 0x86DD     │  │ Other    │                │
│     │ (IPv4)     │  │ (IPv6)     │  │ (Pass)   │                │
│     └───────┬────┘  └──────┬─────┘  └──────────┘                │
│             │              │                                     │
│     ┌───────▼────┐  ┌──────▼─────┐                               │
│     │ Parse IPv4 │  │ Parse IPv6 │                               │
│     │ iphdr      │  │ ipv6hdr    │                               │
│     └───────┬────┘  └──────┬─────┘                               │
│             │              │                                     │
│     ┌───────▼────┐  ┌──────▼─────┐                               │
│     │ VIP Lookup │  │ VIP Lookup │                               │
│     │ vip_map    │  │ vip_map_v6 │                               │
│     └───────┬────┘  └──────┬─────┘                               │
│             │              │                                     │
│             └──────┬───────┘                                     │
│                    │                                             │
│             ┌──────▼──────┐                                      │
│             │ Maglev Hash │                                      │
│             └──────┬──────┘                                      │
│                    │                                             │
│             ┌──────▼──────┐                                      │
│             │ Select Real│                                      │
│             └──────┬──────┘                                      │
│                    │                                             │
│        ┌───────────┼───────────┐                                 │
│        │           │           │                                 │
│  ┌─────▼────┐ ┌────▼────┐ ┌────▼─────┐                          │
│  │ IPv4 Real│ │ IPv6 Real│ │ Mixed   │                          │
│  └─────┬────┘ └────┬────┘ └────┬─────┘                          │
│        │           │           │                                 │
│  ┌─────▼────┐ ┌────▼────┐ ┌────▼─────┐                          │
│  │ IPIP     │ │ ip6ip6  │ │ ip6ip6   │                          │
│  │ Encap    │ │ Encap   │ │ Encap    │                          │
│  └──────────┘ └─────────┘ └──────────┘                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 BPF Map Changes

**Why separate maps?** Performance optimization - smaller maps for IPv4.

```c
// IPv4 VIP Map (existing)
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 64);
    __type(key, u32);                    // 4 bytes
    __type(value, struct vip_meta);
} vip_map SEC(".maps");

// IPv6 VIP Map (new)
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 64);
    __type(key, struct in6_addr);        // 16 bytes
    __type(value, struct vip_meta);
} vip_map_v6 SEC(".maps");

// Real Server Maps
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, u32);
    __type(value, struct real_meta);
} real_map SEC(".maps");

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, struct in6_addr);
    __type(value, struct real_meta);
} real_map_v6 SEC(".maps");

// LRU Connection Tracking (dual-stack)
struct conn_key {
    u8 family;              // AF_INET or AF_INET6
    union {
        u32 ipv4_src;
        struct in6_addr ipv6_src;
    };
    union {
        u32 ipv4_dst;
        struct in6_addr ipv6_dst;
    };
    u16 src_port;
    u16 dst_port;
    u8 proto;
} __attribute__((packed));

struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, 8000000);
    __type(key, struct conn_key);
    __type(value, u32);     // Real server index
} lru_map SEC(".maps");
```

### 4.4 XDP Packet Parser Changes

```c
// Dual-stack packet parsing
SEC("xdp")
int xdp_load_balancer(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    struct ethhdr *eth = data;
    
    // Bounds check
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;
    
    u16 eth_proto = bpf_ntohs(eth->h_proto);
    
    switch (eth_proto) {
    case ETH_P_IP:    // 0x0800 - IPv4
        return process_ipv4(ctx, data, data_end);
        
    case ETH_P_IPV6:  // 0x86DD - IPv6
        return process_ipv6(ctx, data, data_end);
        
    default:
        return XDP_PASS;
    }
}

static __always_inline int process_ipv6(struct xdp_md *ctx,
                                         void *data, void *data_end) {
    struct ethhdr *eth = data;
    struct ipv6hdr *ip6 = (void *)(eth + 1);
    
    // Bounds check
    if ((void *)(ip6 + 1) > data_end)
        return XDP_PASS;
    
    // IPv6 VIP lookup
    struct in6_addr dst = ip6->daddr;
    struct vip_meta *vip = bpf_map_lookup_elem(&vip_map_v6, &dst);
    if (!vip)
        return XDP_PASS;
    
    // Get next header (could be extension header)
    u8 next_hdr = ip6->nexthdr;
    void *l4_hdr = (void *)(ip6 + 1);
    
    // Skip extension headers (simplified)
    if (next_hdr == NEXTHDR_TCP) {
        struct tcphdr *tcp = l4_hdr;
        if ((void *)(tcp + 1) > data_end)
            return XDP_PASS;
        
        // 5-tuple hash for IPv6
        u32 hash = compute_ipv6_hash(&ip6->saddr, &ip6->daddr,
                                      tcp->source, tcp->dest);
        
        // Maglev lookup and encapsulation...
        return encap_ipv6(ctx, ip6, tcp, vip, hash);
    }
    
    return XDP_PASS;
}
```

### 4.5 IPv6 Encapsulation

**Why ip6ip6 mode?** RFC 2473 standard for IPv6-in-IPv6 tunneling.

```c
static __always_inline int encap_ipv6(struct xdp_md *ctx,
                                       struct ipv6hdr *inner_ip6,
                                       struct tcphdr *tcp,
                                       struct vip_meta *vip,
                                       u32 hash) {
    // Get real server
    u32 real_idx = maglev_lookup(vip, hash);
    struct in6_addr *real_addr = bpf_map_lookup_elem(&real_map_v6, &real_idx);
    if (!real_addr)
        return XDP_DROP;
    
    // Adjust headroom for outer IPv6 header (40 bytes)
    if (bpf_xdp_adjust_head(ctx, -sizeof(struct ipv6hdr)))
        return XDP_DROP;
    
    // Re-establish pointers after adjustment
    data = (void *)(long)ctx->data;
    data_end = (void *)(long)ctx->data_end;
    
    struct ethhdr *new_eth = data;
    struct ipv6hdr *outer_ip6 = (void *)(new_eth + 1);
    
    if ((void *)(outer_ip6 + 1) > data_end)
        return XDP_DROP;
    
    // Build outer IPv6 header
    outer_ip6->version = 6;
    outer_ip6->priority = 0;
    outer_ip6->flow_lbl[0] = outer_ip6->flow_lbl[1] = outer_ip6->flow_lbl[2] = 0;
    outer_ip6->payload_len = bpf_ntohs(inner_ip6_len + inner_payload_len);
    outer_ip6->nexthdr = NEXTHDR_IPV6;  // 41 (IPv6-in-IPv6)
    outer_ip6->hop_limit = 64;
    
    // Source: RSS-friendly address
    outer_ip6->saddr = rss_friendly_addr;
    
    // Destination: Real server
    outer_ip6->daddr = *real_addr;
    
    // Copy Ethernet header
    // ... (set destination MAC for real server)
    
    return XDP_TX;
}
```

### 4.6 Tunnel Interface Configuration

```bash
# IPv4 IPIP tunnel (existing)
ip link add name ipip0 type ipip external
ip link set up dev ipip0

# IPv6-in-IPv6 tunnel (new)
ip link add name ipip60 type ip6tnl mode ip6ip6 external
ip link set up dev ipip60

# For IPv4-in-IPv6 (if needed for IPv4 reals with IPv6 transport)
ip link add name ipip61 type ip6tnl mode ipip6 external
ip link set up dev ipip61
```

---

## 5. Silverton IPv6 Adaptation

### 5.1 Why Silverton Needs Changes

Silverton propagates neighbor information from switches to servers. The current implementation:

1. **Monitors ARP table only** - No IPv6 neighbor discovery
2. **Uses AF_INET netlink** - IPv4-only socket family
3. **Sends IPv4 addresses in gRPC** - No address family field

### 5.2 Proposed Solution: Dual-Stack Neighbor Propagation

```
┌─────────────────────────────────────────────────────────────────┐
│                 SILVERTON DUAL-STACK ARCHITECTURE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    ARISTA SWITCH (EOS)                     │  │
│  │                                                            │  │
│  │  ┌─────────────────────┐    ┌─────────────────────┐        │  │
│  │  │   ARP Table Watch   │    │   NDP Table Watch   │        │  │
│  │  │   (IPv4)            │    │   (IPv6)            │        │  │
│  │  │                     │    │                     │        │  │
│  │  │  eos::ArpTable      │    │  eos::NeighborTable │        │  │
│  │  │  handler()          │    │  handler()          │        │  │
│  │  └──────────┬──────────┘    └──────────┬──────────┘        │  │
│  │             │                          │                   │  │
│  │             └────────────┬─────────────┘                   │  │
│  │                          │                                 │  │
│  │               ┌──────────▼──────────┐                      │  │
│  │               │  Silverton Ctrl     │                      │  │
│  │               │  (gRPC Server)      │                      │  │
│  │               └──────────┬──────────┘                      │  │
│  └──────────────────────────┼─────────────────────────────────┘  │
│                             │                                    │
│                             │ gRPC Stream                         │
│                             │ (NeighborUpdate messages)          │
│                             ▼                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    EDGE SERVER (Linux)                     │  │
│  │                                                            │  │
│  │               ┌──────────▼──────────┐                      │  │
│  │               │  Silverton Agent    │                      │  │
│  │               └──────────┬──────────┘                      │  │
│  │                          │                                 │  │
│  │             ┌────────────┼────────────┐                    │  │
│  │             │            │            │                    │  │
│  │  ┌──────────▼───┐ ┌──────▼──────┐ ┌──▼──────────┐         │  │
│  │  │ ARP Entry    │ │ NDP Entry   │ │ Delete Entry│         │  │
│  │  │ AF_INET      │ │ AF_INET6    │ │ (both)      │         │  │
│  │  └──────────┬───┘ └──────┬──────┘ └─────────────┘         │  │
│  │             │            │                                 │  │
│  │             └────────────┼────────────────────────────────┘  │
│  │                          │                                   │
│  │               ┌──────────▼──────────┐                        │
│  │               │  Kernel Neighbor    │                        │
│  │               │  Table (NUD_PERM)   │                        │
│  │               └─────────────────────┘                        │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 gRPC Protocol Extension

**Why add address family?** To distinguish IPv4 and IPv6 neighbors.

```protobuf
// neighbor_update.proto

syntax = "proto3";

package silverton;

enum AddressFamily {
  AF_INET = 0;    // IPv4
  AF_INET6 = 1;   // IPv6
}

enum UpdateType {
  ADD_OR_UPDATE = 0;
  DELETE = 1;
}

message NeighborUpdate {
  // IP address (IPv4 or IPv6 string representation)
  string ip_address = 1;
  
  // MAC address (empty for DELETE)
  string mac_address = 2;
  
  // Interface name
  string interface = 3;
  
  // VRF ID
  int32 vrf = 4;
  
  // Update type
  UpdateType type = 5;
  
  // Address family (NEW - critical for IPv6)
  AddressFamily family = 6;
  
  // Optional: VLAN ID
  int32 vlan_id = 7;
}

service Silverton {
  // Bidirectional streaming for real-time updates
  rpc StreamNeighborUpdates(stream Registration) 
      returns (stream NeighborUpdate);
  
  // One-shot registration
  rpc Register(Registration) returns (RegistrationResponse);
}

message Registration {
  string hostname = 1;
  repeated string interfaces = 2;
  bool enable_ipv4 = 3;
  bool enable_ipv6 = 4;
}
```

### 5.4 Switch-Side Implementation (EOS SDK)

```cpp
// silverton_controller.cpp

class SilvertonController : public eos::sdk_agent {
public:
    SilvertonController(eos::sdk *sdk) 
        : eos::sdk_agent(sdk),
          arp_mgr_(sdk->get_arp_table_mgr()),
          neighbor_mgr_(sdk->get_neighbor_table_mgr()) {
        
        // Register for IPv4 ARP changes
        arp_mgr_->arp_table_is(true);
        
        // Register for IPv6 NDP changes (NEW)
        neighbor_mgr_->neighbor_table_is(true);
    }
    
    void on_arp_entry_set(eos::arp_key_t const& key, 
                          eos::arp_entry_t const& entry) override {
        // IPv4 ARP entry changed
        NeighborUpdate update;
        update.set_ip_address(key.ip_addr().to_string());
        update.set_mac_address(entry.mac_addr().to_string());
        update.set_interface(key.intf().to_string());
        update.set_type(ADD_OR_UPDATE);
        update.set_family(AddressFamily::AF_INET);
        
        broadcast_update(update);
    }
    
    void on_arp_entry_del(eos::arp_key_t const& key) override {
        // IPv4 ARP entry deleted
        NeighborUpdate update;
        update.set_ip_address(key.ip_addr().to_string());
        update.set_type(DELETE);
        update.set_family(AddressFamily::AF_INET);
        
        broadcast_update(update);
    }
    
    // NEW: IPv6 NDP handlers
    void on_neighbor_entry_set(eos::neighbor_key_t const& key,
                                eos::neighbor_entry_t const& entry) override {
        // IPv6 NDP entry changed
        NeighborUpdate update;
        update.set_ip_address(key.ip_addr().to_string());
        update.set_mac_address(entry.mac_addr().to_string());
        update.set_interface(key.intf().to_string());
        update.set_type(ADD_OR_UPDATE);
        update.set_family(AddressFamily::AF_INET6);  // IPv6!
        
        broadcast_update(update);
    }
    
    void on_neighbor_entry_del(eos::neighbor_key_t const& key) override {
        // IPv6 NDP entry deleted
        NeighborUpdate update;
        update.set_ip_address(key.ip_addr().to_string());
        update.set_type(DELETE);
        update.set_family(AddressFamily::AF_INET6);
        
        broadcast_update(update);
    }
    
private:
    void broadcast_update(const NeighborUpdate& update) {
        for (auto& stub : registered_agents_) {
            stub->StreamNeighborUpdates(update);
        }
    }
    
    eos::arp_table_mgr *arp_mgr_;
    eos::neighbor_table_mgr *neighbor_mgr_;  // NEW
    std::vector<std::unique_ptr<Silverton::Stub>> registered_agents_;
};
```

### 5.5 Server-Side Implementation (Linux Agent)

```python
#!/usr/bin/env python3
# silverton_agent.py

import grpc
import netlink
import socket
import neighbor_update_pb2
import neighbor_update_pb2_grpc

class SilvertonAgent:
    def __init__(self, config):
        self.config = config
        self.nl_socket = netlink.socket()
        
    def on_neighbor_update(self, update: neighbor_update_pb2.NeighborUpdate):
        """Process neighbor update from controller."""
        
        if update.type == neighbor_update_pb2.DELETE:
            self.delete_neighbor(update)
        else:
            self.add_neighbor(update)
    
    def add_neighbor(self, update: neighbor_update_pb2.NeighborUpdate):
        """Add static neighbor entry via netlink."""
        
        # Determine address family
        if update.family == neighbor_update_pb2.AF_INET:
            family = socket.AF_INET
            ip_bytes = socket.inet_pton(socket.AF_INET, update.ip_address)
        else:  # AF_INET6
            family = socket.AF_INET6
            ip_bytes = socket.inet_pton(socket.AF_INET6, update.ip_address)
        
        # Create netlink neighbor message
        msg = netlink.NeighborMessage()
        msg.family = family
        msg.ifindex = netlink.if_nametoindex(update.interface)
        msg.state = netlink.NUD_PERMANENT  # Static - never expires
        msg.flags = netlink.NTF_NONE
        
        # Set attributes
        msg.attrs = [
            (netlink.NDA_DST, ip_bytes),           # IP address
            (netlink.NDA_LLADDR, update.mac_address),  # MAC address
        ]
        
        # Send to kernel
        flags = netlink.NLM_F_REQUEST | netlink.NLM_F_CREATE | netlink.NLM_F_REPLACE
        netlink.nl_socket.send(msg, flags)
        
        logging.info(f"Added {'IPv6' if update.family else 'IPv4'} neighbor: "
                    f"{update.ip_address} -> {update.mac_address}")
    
    def delete_neighbor(self, update: neighbor_update_pb2.NeighborUpdate):
        """Delete neighbor entry via netlink."""
        
        if update.family == neighbor_update_pb2.AF_INET:
            family = socket.AF_INET
            ip_bytes = socket.inet_pton(socket.AF_INET, update.ip_address)
        else:
            family = socket.AF_INET6
            ip_bytes = socket.inet_pton(socket.AF_INET6, update.ip_address)
        
        msg = netlink.NeighborMessage()
        msg.family = family
        msg.ifindex = netlink.if_nametoindex(update.interface)
        msg.attrs = [(netlink.NDA_DST, ip_bytes)]
        
        netlink.nl_socket.send(msg, netlink.NLM_F_REQUEST | netlink.NLM_F_ACK)
        
        logging.info(f"Deleted neighbor: {update.ip_address}")
    
    def run(self):
        """Main loop: receive updates from controller."""
        
        for controller in self.config['controllers']:
            try:
                channel = grpc.insecure_channel(controller)
                stub = neighbor_update_pb2_grpc.SilvertonStub(channel)
                
                # Register for both IPv4 and IPv6
                registration = neighbor_update_pb2.Registration(
                    hostname=os.uname().nodename,
                    interfaces=[self.config['interface']],
                    enable_ipv4=True,
                    enable_ipv6=True
                )
                
                # Stream updates
                for update in stub.StreamNeighborUpdates(registration):
                    self.on_neighbor_update(update)
                    
            except grpc.RpcError as e:
                logging.error(f"gRPC error: {e}")
                time.sleep(self.config['retry_interval'])
```

### 5.6 Verification Commands

```bash
# Check IPv4 ARP entries
ip neigh show nud permanent

# Check IPv6 NDP entries
ip -6 neigh show nud permanent

# Example output:
# 10.0.0.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff nud permanent
# 2001:db8:0:1::1 dev eth0 lladdr aa:bb:cc:dd:ee:ff nud permanent

# Manual test (IPv6)
ip -6 neigh replace 2001:db8:0:1::1 lladdr aa:bb:cc:dd:ee:ff dev eth0 nud permanent

# Check Silverton agent logs
journalctl -u silverton-agent -f
```

---

## 6. BIRD Configuration for IPv6

### 6.1 Why Separate BGP Sessions

IPv4 and IPv6 typically use separate BGP sessions (or MP-BGP with address family negotiation). This provides:

1. **Independent peering** - Each family can have different policies
2. **Clear separation** - Easier troubleshooting
3. **Compatibility** - Works with all BGP implementations

### 6.2 Switch BIRD Configuration

```bash
# /etc/bird/bird.conf on Arista switch

# ============================================
# IPv4 Configuration (existing)
# ============================================

# eBGP with transit providers
protocol bgp transit1_v4 {
    local as 64512;
    neighbor 10.0.0.1 as 64513;
    import all;
    export all;
    
    # IPv4 address family (implicit)
}

protocol bgp transit2_v4 {
    local as 64512;
    neighbor 10.0.0.2 as 64514;
    import all;
    export all;
}

# iBGP route reflection to hosts
protocol bgp host_rr_v4 {
    local as 64512;
    neighbor 10.0.2.1 as 64512;  # Edge-1
    neighbor 10.0.2.2 as 64512;  # Edge-2
    neighbor 10.0.2.3 as 64512;  # Edge-3
    neighbor 10.0.2.4 as 64512;  # Edge-4
    neighbor 10.0.2.5 as 64512;  # Edge-5
    neighbor 10.0.2.6 as 64512;  # Edge-6
    
    import none;
    export all;
    rr client;
}

# ============================================
# IPv6 Configuration (new)
# ============================================

# eBGP with transit providers (IPv6)
protocol bgp transit1_v6 {
    local as 64512;
    neighbor 2001:db8:0:1::1 as 64513;
    import all;
    export all;
    
    # IPv6 address family
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

# iBGP route reflection to hosts (IPv6)
protocol bgp host_rr_v6 {
    local as 64512;
    neighbor 2001:db8:2::1 as 64512;  # Edge-1
    neighbor 2001:db8:2::2 as 64512;  # Edge-2
    neighbor 2001:db8:2::3 as 64512;  # Edge-3
    neighbor 2001:db8:2::4 as 64512;  # Edge-4
    neighbor 2001:db8:2::5 as 64512;  # Edge-5
    neighbor 2001:db8:2::6 as 64512;  # Edge-6
    
    import none;
    export all;
    rr client;
    
    ipv6 {
        import none;
        export all;
    };
}
```

### 6.3 Host BIRD Configuration

```bash
# /etc/bird/bird.conf on each Edge Server

# ============================================
# IPv4 Configuration (existing)
# ============================================

protocol bgp switch_rr_v4 {
    local as 64512;
    neighbor 10.0.1.254 as 64512;  # Switch IPv4
    import all;
    export none;
}

protocol kernel {
    scan time 60;
    import none;
    export all;  # Export BGP routes to kernel
}

protocol static {
    route 203.0.113.10/32 via "lo";  # IPv4 VIP
}

# ============================================
# IPv6 Configuration (new)
# ============================================

protocol bgp switch_rr_v6 {
    local as 64512;
    neighbor 2001:db8:1::254 as 64512;  # Switch IPv6
    import all;
    export none;
    
    ipv6 {
        import all;
        export none;
    };
}

protocol kernel {
    scan time 60;
    import none;
    export all;
    
    ipv6 {
        import none;
        export all;  # Export IPv6 routes to kernel
    };
}

protocol static {
    ipv6;
    route 2001:db8::10/128 via "lo";  # IPv6 VIP
}
```

### 6.4 Route Table Verification

```bash
# Check IPv4 routes
ip route show | wc -l
# Expected: ~800000 routes

# Check IPv6 routes
ip -6 route show | wc -l
# Expected: ~100000 routes

# BIRD status
birdc show route table all count
birdc show route protocol switch_rr_v4
birdc show route protocol switch_rr_v6
```

---

## 7. Traffic Steering IPv6 Support

### 7.1 Current State

The Traffic Steering system already has partial IPv6 support:

- **IPSets**: `egress_path_0_v6` and `egress_path_1_v6` defined
- **nftables**: `ip6 daddr` rules present
- **Missing**: IPv6 eBPF maps for statistics

### 7.2 eBPF Map Changes

```c
// tcp_monitor.bpf.c

// IPv4 stats map (existing)
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, 65536);
    __type(key, u32);  // IPv4 destination
    __type(value, struct dest_stats);
} dest_stats_map SEC(".maps");

// IPv6 stats map (new)
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, 65536);
    __type(key, struct in6_addr);  // IPv6 destination
    __type(value, struct dest_stats);
} dest_stats_map_v6 SEC(".maps");

// Unified tracepoint handler
SEC("tracepoint/tcp/tcp_probe")
int BPF_PROG(trace_tcp_probe, const struct sock *sk) {
    struct inet_sock *inet = (struct inet_sock *)sk;
    
    // Get address family
    u8 family = BPF_CORE_READ(sk, __sk_common.skc_family);
    
    if (family == AF_INET) {
        u32 daddr = BPF_CORE_READ(inet, inet_daddr);
        update_stats_v4(daddr, sk);
    } else if (family == AF_INET6) {
        struct in6_addr daddr;
        BPF_CORE_READ_INTO(&daddr, sk, __sk_common.skc_v6_daddr);
        update_stats_v6(&daddr, sk);
    }
    
    return 0;
}

static __always_inline void update_stats_v6(struct in6_addr *daddr, 
                                             const struct sock *sk) {
    struct tcp_sock *tp = (struct tcp_sock *)sk;
    u32 srtt_us = BPF_CORE_READ(tp, srtt_us) >> 3;
    
    struct dest_stats new_stats = {};
    struct dest_stats *stats = bpf_map_lookup_or_try_init(
        &dest_stats_map_v6, daddr, &new_stats);
    
    if (stats) {
        stats->rtt_us = srtt_us;
        stats->last_update = bpf_ktime_get_ns();
    }
}
```

### 7.3 steerd Configuration Update

```yaml
# /etc/steerd/config.yaml

# Add IPv6 path configuration
paths:
  - id: 0
    name: "primary"
    interface: "eth0"
    ipset_v4: "egress_path_0_v4"
    ipset_v6: "egress_path_0_v6"  # Already defined
    fwmark: 0x100
    snat_pool:
      - 198.51.100.2
      - 198.51.100.3
      # ... IPv4 SNAT
    snat_pool_v6:                  # NEW
      - "2001:db8:100::2"
      - "2001:db8:100::3"
      - "2001:db8:100::4"
      - "2001:db8:100::5"
      - "2001:db8:100::6"
      - "2001:db8:100::7"
  
  - id: 1
    name: "failover"
    interface: "eth1"
    ipset_v4: "egress_path_1_v4"
    ipset_v6: "egress_path_1_v6"
    fwmark: 0x101
    snat_pool:
      - 203.0.113.2
      # ... IPv4 SNAT
    snat_pool_v6:                  # NEW
      - "2001:db8:200::2"
      - "2001:db8:200::3"
      - "2001:db8:200::4"
      - "2001:db8:200::5"
      - "2001:db8:200::6"
      - "2001:db8:200::7"
```

### 7.4 nftables IPv6 SNAT

```bash
# /etc/nftables/traffic-steering.nft

table inet traffic_steering {
    # IPv6 destination sets
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
        
        # IPv4 marking (existing)
        ip daddr @egress_path_0_v4 meta mark set 0x100
        ip daddr @egress_path_1_v4 meta mark set 0x101
        
        # IPv6 marking (existing)
        ip6 daddr @egress_path_0_v6 meta mark set 0x100
        ip6 daddr @egress_path_1_v6 meta mark set 0x101
    }
    
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        
        # IPv4 SNAT (existing)
        oif "eth0" meta mark 0x100 snat to 198.51.100.2-198.51.100.7
        oif "eth1" meta mark 0x101 snat to 203.0.113.2-203.0.113.7
        
        # IPv6 SNAT (new)
        oif "eth0" meta mark 0x100 snat ip6 to 2001:db8:100::2-2001:db8:100::7
        oif "eth1" meta mark 0x101 snat ip6 to 2001:db8:200::2-2001:db8:200::7
    }
}
```

---

## 8. Implementation Checklist

### Phase 1: Katran IPv6 Support

- [ ] **BPF Map Updates**
  - [ ] Add `vip_map_v6` with `struct in6_addr` key
  - [ ] Add `real_map_v6` with `struct in6_addr` key
  - [ ] Update LRU map to support dual-stack keys
  - [ ] Test map operations with IPv6 addresses

- [ ] **XDP Parser Updates**
  - [ ] Add `process_ipv6()` function
  - [ ] Handle IPv6 extension headers
  - [ ] Implement IPv6 5-tuple hash
  - [ ] Add IPv6 flow label optimization

- [ ] **Encapsulation Updates**
  - [ ] Implement `ip6ip6` encapsulation
  - [ ] Configure `ipip60` tunnel interface
  - [ ] Test IPv6-in-IPv6 tunneling

- [ ] **Edge Server Configuration**
  - [ ] Add IPv6 VIP to loopback
  - [ ] Configure IPv6 IPIP decapsulation
  - [ ] Disable IPv6 rp_filter

### Phase 2: Silverton IPv6 Support

- [ ] **Switch-Side (EOS)**
  - [ ] Add `eos::NeighborTable` monitoring
  - [ ] Implement NDP change handlers
  - [ ] Update gRPC protocol with address family
  - [ ] Test NDP propagation

- [ ] **Server-Side (Agent)**
  - [ ] Add `AF_INET6` netlink support
  - [ ] Implement IPv6 neighbor addition
  - [ ] Implement IPv6 neighbor deletion
  - [ ] Test static NDP entries

- [ ] **gRPC Protocol**
  - [ ] Add `AddressFamily` enum
  - [ ] Update `NeighborUpdate` message
  - [ ] Backward compatibility testing

### Phase 3: BIRD IPv6 Configuration

- [ ] **Switch BIRD**
  - [ ] Add IPv6 eBGP sessions with transits
  - [ ] Add IPv6 iBGP route reflection
  - [ ] Test IPv6 route propagation

- [ ] **Host BIRD**
  - [ ] Add IPv6 iBGP session with switch
  - [ ] Configure IPv6 kernel protocol
  - [ ] Add IPv6 VIP static route
  - [ ] Verify IPv6 routes in kernel

### Phase 4: Traffic Steering IPv6

- [ ] **eBPF Maps**
  - [ ] Add `dest_stats_map_v6`
  - [ ] Update tracepoint handlers
  - [ ] Test IPv6 stats collection

- [ ] **steerd Configuration**
  - [ ] Add IPv6 SNAT pools
  - [ ] Update path configuration
  - [ ] Test IPv6 path selection

- [ ] **nftables**
  - [ ] Add IPv6 SNAT rules
  - [ ] Test IPv6 marking and NAT

### Phase 5: Integration Testing

- [ ] **Dual-Stack Testing**
  - [ ] Test IPv4-only traffic
  - [ ] Test IPv6-only traffic
  - [ ] Test mixed IPv4/IPv6 traffic

- [ ] **Performance Testing**
  - [ ] Benchmark IPv4 PPS
  - [ ] Benchmark IPv6 PPS
  - [ ] Compare and optimize

- [ ] **Failover Testing**
  - [ ] Test IPv6 path failover
  - [ ] Test IPv6 route convergence
  - [ ] Test NDP propagation timing

---

## 9. Why These Solutions Were Chosen

### 9.1 Katran: Dual-Stack XDP with Separate Maps

**Decision**: Use separate BPF maps for IPv4 and IPv6 instead of unified maps.

**Rationale**:
1. **Performance**: Smaller IPv4 maps mean better cache locality
2. **Memory Efficiency**: Don't waste 16-byte keys for 4-byte addresses
3. **Backward Compatibility**: Existing IPv4 configuration unchanged
4. **Incremental Migration**: Can deploy IPv6 without touching IPv4

**Alternative Considered**: Unified maps with address family in key.
- Rejected due to 4x memory overhead for IPv4 entries

### 9.2 Silverton: NDP Monitoring with Address Family Field

**Decision**: Add NDP monitoring alongside ARP, with explicit address family in gRPC.

**Rationale**:
1. **Protocol Separation**: ARP and NDP are fundamentally different protocols
2. **Clear Semantics**: Address family field removes ambiguity
3. **EOS SDK Support**: `eos::NeighborTable` already exists for IPv6
4. **Backward Compatible**: New field is optional for existing IPv4 clients

**Alternative Considered**: Single unified neighbor table.
- Rejected because EOS SDK separates ARP and NDP tables

### 9.3 BIRD: Separate IPv4 and IPv6 Sessions

**Decision**: Use separate BGP sessions for each address family.

**Rationale**:
1. **Simplicity**: Clear configuration, easier debugging
2. **Compatibility**: Works with all BGP implementations
3. **Independent Policies**: Can apply different import/export rules
4. **Graceful Degradation**: One family can fail without affecting the other

**Alternative Considered**: MP-BGP with both families in one session.
- Rejected for operational simplicity and wider compatibility

### 9.4 Traffic Steering: Parallel IPv6 Infrastructure

**Decision**: Mirror IPv4 infrastructure for IPv6 (separate maps, IPSets, SNAT pools).

**Rationale**:
1. **Consistency**: Same operational model for both families
2. **Independent Scaling**: IPv6 can grow without affecting IPv4
3. **Clear Separation**: Easier to debug and monitor
4. **Performance**: Smaller maps per family

**Alternative Considered**: Unified maps with 16-byte keys.
- Rejected for same reasons as Katran

### 9.5 Encapsulation: ip6ip6 Mode

**Decision**: Use IPv6-in-IPv6 (protocol 41) for IPv6 traffic.

**Rationale**:
1. **RFC 2473 Compliance**: Standard IPv6 tunneling
2. **End-to-End**: No translation needed
3. **Performance**: Direct encapsulation, no intermediate headers
4. **Compatibility**: Supported by all IPv6 implementations

**Alternative Considered**: GRE over IPv6.
- Rejected for additional header overhead (4 bytes)

### 9.6 Hash Optimization: IPv6 Flow Label

**Decision**: Use IPv6 flow label as pre-computed hash when available.

**Rationale**:
1. **RFC 6437**: Flow label designed for this purpose
2. **Performance**: Avoid hashing 256 bits of addresses
3. **Stateless**: No per-connection state needed
4. **Backward Compatible**: Fall back to full hash if flow label is zero

**Alternative Considered**: Always hash full 5-tuple.
- Rejected for performance impact on high-PPS scenarios

---

## Summary

This IPv6 adaptation follows industry best practices:

| Principle | Implementation |
|-----------|----------------|
| **Dual-Stack** | IPv4 and IPv6 operate in parallel |
| **Performance** | Separate maps, flow label optimization |
| **Compatibility** | Backward compatible with existing IPv4 |
| **Standards** | RFC-compliant NDP, ip6ip6, BGP |
| **Operations** | Clear separation for debugging |

The proposed changes enable full IPv6 support while maintaining the high performance and scalability characteristics of the original architecture.
