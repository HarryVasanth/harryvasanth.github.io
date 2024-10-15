---
layout: post
title: "Networking - Tailscale: Advanced Subnet Routing and Site-to-Site Integration"
date: 2024-10-15 22:37:27 +01
categories: networking security
tags: tailscale networking vpn zerotrust linux administration
---

## The Hybrid Cloud Connectivity Challenge

We have discussed the basics of Tailscale and Zero Trust, but a experienced administrator often faces a more complex challenge: **Legacy Connectivity**. You have a cloud VPC in AWS, a physical datacenter with 500 servers that don't have Tailscale installed, and a remote team that needs to access all of it seamlessly.

You cannot install a Tailscale agent on every single legacy database, printer, or industrial controller. The solution is **Subnet Routing**. This allows a single Linux box to act as a "Transparent Gateway" between your Tailscale mesh and your traditional physical networks.

## Architecture: The Subnet Router as a Bridge

A Subnet Router is a Tailscale-enabled node that advertises a range of "real" IP addresses (e.g., `10.0.0.0/24`) to the Tailnet. Other nodes on the Tailnet see those IPs and automatically route traffic for them through the gateway.

## Implementation: Setting up a Subnet Router

### 1. Enable IP Forwarding on Linux

Before Tailscale can route traffic, the Linux kernel must be told to allow it.

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### 2. Advertise the Routes

When starting Tailscale, we tell it which subnets this node "owns."

```bash
sudo tailscale up --advertise-routes=10.0.1.0/24,192.168.10.0/24
```

### 3. Approve the Routes in the Console

For security, Tailscale will not start routing until you manually approve these routes in the Tailscale Admin Console (Edit Route Settings -> Enable All).

## Phase 2: High Availability with Failover

What if your Subnet Router fails? Your entire remote team loses access to the datacenter.
**The Ideal Fix**: Deploy **High Availability (HA) Subnet Routers**.
If you run the exact same `advertise-routes` command on _two_ different servers, Tailscale will automatically pick one as the primary. If the primary goes offline, the Tailscale coordination server will automatically update all clients to use the secondary gateway within seconds.

## Phase 3: Site-to-Site Integration (The 'LAN-to-LAN' Secret)

To make this a true site-to-site VPN, you need the physical devices _behind_ the Subnet Router to be able to find the Tailscale nodes.

1.  **Static Routes**: On your physical router (Mikrotik/OPNsense), add a static route for the Tailscale range (usually `100.64.0.0/10`) pointing to the internal IP of the Subnet Router.
2.  **MSS Clamping**: Because Tailscale/WireGuard adds overhead, you **must** implement MSS Clamping on the Subnet Router to prevent large packets from being dropped.

```bash
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

## Why This Wins Over Legacy IPsec

- **No NAT Issues**: Tailscale's DERP system punches through even the most restrictive double-NAT environments.
- **Performance**: Based on WireGuard, it is significantly faster and uses less CPU than AES-CBC based IPsec tunnels.
- **Visibility**: You can see every active connection and audit every ACL change in a single web interface.

## Summary

Subnet Routing is the bridge between the future of Zero Trust and the reality of legacy infrastructure. By implementing HA Subnet Routers and proper site-to-site routing, you provide your organisation with a connectivity fabric that is invisible to the user but robust and secure for the administrator. It is the ultimate tool for managing the complexity of the modern hybrid-cloud network.
