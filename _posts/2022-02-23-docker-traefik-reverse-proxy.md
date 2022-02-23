---
layout: post
title: "Docker - Traefik: Resolving 'Client IP' pass-through in NAT and Proxy Environments"
date: 2022-02-23 11:25:27 +01
categories: docker networking
tags: traefik proxy docker-networking networking devops
---

## The Problem: IP Address Obfuscation

You've set up Traefik as a reverse proxy for your Docker containers. However, in your application logs (Nginx, Go, PHP), every visitor appears to be coming from the same IP: `172.18.0.1` (the Docker Gateway). This breaks Geo-IP blocking, rate-limiting, and security auditing.

## The Optimal Solution: Preserving the Source IP

There are two primary ways to fix this, depending on your network architecture.

### Solution 1: Host Mode Networking (Simplest)

By default, Docker uses a userland proxy that performs SNAT (Source Network Address Translation). By running Traefik in `host` mode, you bypass the Docker bridge entirely for the entrypoint.

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v2.10
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host # This is the magic line
```

### Solution 2: PROXY Protocol (For Cloud/Hardware LBs)

If your Traefik instance is behind _another_ load balancer (like AWS NLB or HAProxy), you should use the **PROXY protocol**. This adds a header at the TCP level containing the original client IP.

1.  **Configure the Entrypoint in Traefik**:
    ```yaml
    --entrypoints.web.proxyProtocol.trustedIPs=127.0.0.1,10.0.0.0/8
    ```
2.  **Configure Nginx/App to trust Traefik**:
    In Nginx, use the `real_ip` module:
    ```conf
    set_real_ip_from  172.18.0.0/16;
    real_ip_header    X-Forwarded-For;
    ```

## Tips & Tricks

- **The 'Internal' vs 'External' Flag**: Traefik 2.x added the ability to define if an entrypoint is trusted. Never set `trustedIPs` to `0.0.0.0/0`, or an attacker can spoof their own `X-Forwarded-For` header and bypass your security rules.
- **Docker Swarm Complexity**: In a Swarm environment, you often hit the "Ingress Mesh" which also performs SNAT. You may need to bypass the ingress mesh and use `mode: host` on every node to get the real IP.
- **Log Forwarding**: If you are sending logs to an ELK or Splunk stack, verify that the `ClientIP` field in the Traefik access log matches the real visitor before you build your dashboards.

## Summary

Seeing the real client IP is not a luxury; it's a security requirement. Whether through `host` mode or the PROXY protocol, ensuring Traefik passes this data correctly is a hallmark of a production-ready container environment.
