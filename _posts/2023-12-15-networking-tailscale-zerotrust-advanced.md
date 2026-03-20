---
layout: post
title: "Networking - Security: Implementing Private Zero Trust Fabrics with Tailscale and ACL Orchestration"
date: 2023-12-28 02:06:09 +01
categories: networking security
tags: tailscale zerotrust wireguard networking vpn security
---

## The Death of the 'Castle and Moat' Model

Traditional networking relies on the "Castle and Moat" security model: everything inside the office firewall is "trusted," and everything outside is "untrusted." In a world of remote work and multi-cloud environments, this model has failed. If an attacker breaches your VPN, they have lateral access to everything in your datacenter.

**Zero Trust Networking** assumes that the network is _always_ hostile. Access is granted based on identity (who are you?) and device health, not on "where" you are connected. **Tailscale** (built on top of WireGuard) has emerged as the leading tool for implementing this in a professional environment.

## How Tailscale Simplifies the Complex

Tailscale handles the "hard parts" of WireGuard that usually require significant manual effort:

1.  **Key Exchange**: Automated via your existing Identity Provider (Google, Microsoft, Okta).
2.  **NAT Traversal**: Using STUN/DERP to connect two servers even if both are behind strict firewalls.
3.  **ACLs (Access Control Lists)**: Centralised, declarative policy that defines who can talk to what.

## Implementation: The 'ACL-First' Network

Instead of opening ports on firewalls, you define your network policy in a central JSON/HuJSON file.

```json
{
  "groups": {
    "group:devs": ["alice@example.com", "bob@example.com"],
    "group:ops": ["charlie@example.com"]
  },
  "tagOwners": {
    "tag:prod-db": ["group:ops"],
    "tag:web-server": ["group:ops", "group:devs"]
  },
  "acls": [
    // Ops can access everything
    { "action": "accept", "src": ["group:ops"], "dst": ["*:*"] },

    // Devs can only access web servers on port 80/443
    {
      "action": "accept",
      "src": ["group:devs"],
      "dst": ["tag:web-server:80", "tag:web-server:443"]
    }
  ]
}
```

## Suitable Strategy: Subnet Routers and High-Availability Exit Nodes

You don't need to install Tailscale on every legacy device.

- **Subnet Router**: A single Linux server inside your VPC can "announce" the whole VPC subnet (e.g., `10.0.1.0/24`) to the Tailnet. Remote users can then hit those internal IPs as if they were local.
- **Exit Node**: You can route all your internet traffic through a specific server in a different country, providing a corporate-controlled breakout point for remote staff.
- **High Availability**: By running two subnet routers for the same range, Tailscale provides automatic failover.

## Why This is a Ideal Choice

1.  **Identity-Based**: If an employee leaves the company and you disable their Okta/Google account, their network access is revoked instantly across every server in the world.
2.  **No Public Ports**: Your servers can have **zero** open ports on the public internet. They only listen on the `tailscale0` interface.
3.  **Cross-Cloud Simplicity**: Connecting an AWS VPC, a Hetzner dedicated server, and a Proxmox VM in your basement takes minutes and requires zero routing configuration.

## Troubleshooting Tailscale Performance

- **DERP Check**: Use `tailscale status` to see if you are using a direct connection or a relay (DERP). If relaying, check your firewall for UDP 41641.
- **Tailscale Funnel**: Use this to expose a local service to the public internet securely with a valid SSL certificate and no port forwarding.

## Summary

Tailscale is not "just a VPN"; it's a programmable network fabric. It allows you to build a secure, global infrastructure that is transparent to the user but strictly controlled by the administrator. In the modern era of infrastructure management, it is the most efficient way to achieve high security without the operational overhead of legacy IPsec clusters. It represents the definitive standard for the "Identity-Defined Perimeter."
