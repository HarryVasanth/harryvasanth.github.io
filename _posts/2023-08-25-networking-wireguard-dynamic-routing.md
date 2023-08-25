---
layout: post
title: "Networking - WireGuard: Implementing Scalable Hub-and-Spoke Networks with Dynamic OSPF Routing"
date: 2023-08-25 02:16:17 +01
categories: networking vpn
tags: wireguard vpn networking routing linux ospf bgp
---

## The Scalability Wall of Static VPNs

WireGuard is widely celebrated for its simplicity, security, and raw performance. However, as an organisation grows, a common architectural failure occurs: the "Static Route Explosion." In a standard setup, every time you add a new subnet to a branch office, you must manually update the `AllowedIPs` and the system routing table on every other node in the network. This "Full Mesh" approach is unmanageable beyond three or four sites.

A well-made network architecture moves away from static management and toward a **Hub-and-Spoke** model combined with **Dynamic Routing (OSPF or BGP)**. In this design, WireGuard acts as a secure, transparent "Virtual Wire" (Layer 3), while a dedicated routing daemon handles the intelligence of route discovery and failover.

## The Architecture: The Hub as a Transit Gateway

1.  **The Hub**: A central Linux server (or OPNsense box) with a static public IP. It acts as the coordination point for all spokes.
2.  **The Spokes**: Remote offices, home labs, or cloud VPCs. They initiate the connection to the Hub.
3.  **The Routing Engine**: We run **FRRouting (FRR)** on all nodes. FRR is a high-performance, modular routing suite that provides industry-standard implementations of OSPF and BGP.

## Implementation: Setting up the 'Virtual Wire'

To allow the routing daemon to handle the traffic, we must configure WireGuard to stay out of the system routing table. We use `Table = off` in the `wg0.conf`.

```conf
# /etc/wireguard/wg0.conf (Hub Example)
[Interface]
Address = 10.255.0.1/32
ListenPort = 51820
PrivateKey = <HUB_PRIVATE_KEY>
# CRITICAL: Prevent WireGuard from adding routes automatically
Table = off

[Peer]
# Spoke 1
PublicKey = <SPOKE1_PUBLIC_KEY>
# Allow all traffic through the tunnel level; the OS will filter via routing table
AllowedIPs = 0.0.0.0/0, ::/0
```

## Adding the Intelligence: OSPF with FRR

Once the secure tunnel is established, we configure FRRouting to find neighbours and exchange routes over the `wg0` interface.

```text
# /etc/frr/frr.conf
router ospf
 ospf router-id 10.255.0.1
 # Advertise our local management subnet
 network 192.168.1.0/24 area 0
 # Advertise the tunnel network
 network 10.255.0.0/24 area 0

interface wg0
 ip ospf area 0
 # Use point-to-multipoint for Hub-and-Spoke environments
 ip ospf network point-to-multipoint
```

**Why Point-to-Multipoint?**: This is the secret for scalable Hub-and-Spoke. It tells OSPF that while all spokes can see the Hub, they cannot necessarily see each other directly. The Hub will relay the routing updates between them, ensuring that Spoke A learns about Spoke B's subnets automatically.

## Suitable Strategy: MTU and MSS Clamping

Because WireGuard adds a 60-80 byte overhead to every packet, and routing protocols like OSPF use large Database Description (DBD) packets, you **must** align your MTU.

- **The Problem**: A standard 1500-byte packet + 80-byte WireGuard header = 1580 bytes. This packet will be dropped by most internet routers.
- **The Fix**: Set `MTU = 1420` on all WireGuard interfaces and implement **MSS Clamping** on the Hub to protect downstream TCP sessions.

```bash
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

## Monitoring the Adjacency

A experienced administrator monitors the routing table, not just the tunnel status.

```bash
# Check if OSPF is established
vtysh -c "show ip ospf neighbor"

# Check the learned routes
ip route show | grep proto ospf
```

If the neighbor state is stuck in `ExStart`, you almost certainly have an MTU mismatch.

## Summary

WireGuard provides the secure encryption, but a professional network requires a routing engine to provide the intelligence. By combining WireGuard with FRRouting, you build a "Corporate SD-WAN" that is as flexible and powerful as expensive commercial solutions, but built on open standards and high-performance code. This is the definitive architecture for connecting distributed infrastructure in 2023. It represents the move from "Managing Tunnels" to "Governing a Global Fabric."
