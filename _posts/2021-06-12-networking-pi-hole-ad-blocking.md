---
layout: post
title: "Networking - Pi-hole & Unbound: Implementing a Private Recursive DNS Resolver"
date: 2021-06-12 20:20:21 +01
categories: networking dns
tags: pi-hole unbound dns security privacy
---

## The Problem: Upstream Trust

Even if you use Pi-hole to block ads, you are still sending your entire DNS history to an upstream provider like Google (8.8.8.8) or Cloudflare (1.1.1.1). They know every site you visit. Furthermore, these providers can (and do) filter results based on their own policies.

## The Optimal Solution: Recursive DNS with Unbound

Instead of "forwarding" requests to someone else, we can run **Unbound** alongside Pi-hole. Unbound will talk directly to the Root DNS servers and handle the entire recursion process ourselves.

### 1. Installing Unbound

```bash
sudo apt update && sudo apt install unbound
```

### 2. Hardening the Configuration

Edit `/etc/unbound/unbound.conf.d/pi-hole.conf`. We want Unbound to listen on a non-standard port (e.g., 5335) so it doesn't conflict with Pi-hole.

```conf
server:
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # Security: Root hints
    root-hints: "/var/lib/unbound/root.hints"

    # Privacy: Hide identity
    hide-identity: yes
    hide-version: yes

    # Performance: Pre-fetching
    prefetch: yes
    num-threads: 1
```

{: file='/etc/unbound/unbound.conf.d/pi-hole.conf'}

### 3. Configuring Pi-hole

In the Pi-hole Web UI, go to **Settings -> DNS**. Uncheck all standard providers and under **Custom 1 (IPv4)**, enter `127.0.0.1#5335`.

## Troubleshooting Key Considerations

- **DNSSEC Validation**: Unbound performs DNSSEC validation by default. If your server's clock is wrong, _all_ DNS lookups will fail. Ensure NTP is working.
- **The Root Hints File**: The IP addresses of the 13 root servers rarely change, but you should update your `root.hints` file once or twice a year:
  ```bash
  wget https://www.internic.net/domain/named.root -O /var/lib/unbound/root.hints
  ```
- **EDNS Client Subnet (ECS)**: Standard resolvers often send part of your IP address to upstream servers to help with "Geo-DNS." By running Unbound yourself, you stop this leak, improving privacy but potentially slightly increasing latency for some CDNs.

## Summary

Combining Pi-hole with Unbound moves you from "Ad-blocking" to "DNS Sovereignty." You no longer rely on a third party to resolve your queries, significantly increasing both your privacy and the security of your network.
