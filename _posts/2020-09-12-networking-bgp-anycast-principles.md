---
layout: post
title: "Networking - BGP Anycast: Principles of Resilient Service Delivery"
date: 2020-09-12 14:48:39 +01
categories: networking linux
tags: bgp anycast dns networking linux routing high-availability
---

## The Problem with Traditional Failover

In a standard network setup, a service has a unique IP address. If the server hosting that IP fails, you must update DNS or a load balancer to point to a new IP. This introduces latency and a single point of failure. Even with HAProxy or Keepalived, you are often limited to a single Layer 2 segment.

**BGP Anycast** flips this logic. Multiple, physically distinct servers all announce the **exact same IP address** to the network via BGP (Border Gateway Protocol). The network's routers then guide each user to the "closest" server based on the shortest BGP path. If one server goes offline, the BGP session drops, and the network automatically re-routes traffic to the next closest instance-usually in under a second.

This is the technology that powers the world's root DNS servers and CDNs like Cloudflare. For an infrastructure administrator, implementing Anycast on a private network or between datacenters provides a level of resilience that no load balancer can match.

## Architecture of an Anycast Node

A professional Anycast node consists of three components:

1.  **The Service**: e.g., BIND or Unbound for DNS.
2.  **The BGP Daemon**: Usually **BIRD** or **FRRouting (FRR)** on Linux.
3.  **The Health Checker**: A script that monitors the service and tells the BGP daemon to stop announcing the IP if the service is unhealthy.

## Implementation: Configuring BIRD for Anycast

Imagine our Anycast IP is `192.0.2.1`. We assign this to the **loopback interface** (`lo`) of every server in our fleet.

### 1. The Linux Interface

```bash
# Add the anycast IP to loopback
ip addr add 192.0.2.1/32 dev lo
```

### 2. The BIRD Configuration

We configure BIRD to peer with our upstream switches and announce the loopback address.

```text
# /etc/bird/bird.conf
protocol device {}

# Monitor the loopback interface
protocol direct {
    interface "lo";
}

# The BGP peering session
protocol bgp my_upstream {
    local as 65001;
    neighbor 10.0.0.1 as 65001;
    ipv4 {
        export filter {
            # ONLY announce our specific anycast IP
            if net = 192.0.2.1/32 then accept;
            reject;
        };
        import all;
    };
}
```

## The Critical Component: Health Checking

If the DNS service crashes but BIRD keeps running, the server will still attract traffic but will be unable to answer queries. This is an "Anycast Black Hole."

**The Ideal Fix**: Use a health check daemon like `anycast-healthchecker`.
It continuously pings the local DNS service. If it fails, the script removes the Anycast IP from the interface or signals BIRD to withdraw the route.

## Advantages for Infrastructure Teams

1.  **Seamless Maintenance**: To upgrade a server, you simply stop the BGP daemon. Traffic drains away to other nodes automatically. Once the upgrade is done, start BGP, and the server rejoins the fleet.
2.  **DDoS Mitigation**: Anycast naturally spreads the load of a volumetric attack across your entire global infrastructure, preventing any single site from being overwhelmed.
3.  **Geographic Performance**: Users in London hit the London node; users in Tokyo hit the Tokyo node-all using the same IP.

## Summary

BGP Anycast is the pinnacle of network-level high availability. It removes the reliance on fragile load balancers and puts the resilience of your services directly into the global routing table. While complex to set up, it provides a level of stability and performance that is essential for mission-critical infrastructure like DNS and global web platforms. In 2020, this is the architecture of choice for high-scale, distributed services.
