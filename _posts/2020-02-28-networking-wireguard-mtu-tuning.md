---
layout: post
title: "Networking - WireGuard: A Comprehensive Guide to MTU, MSS Clamping, and Path Discovery"
date: 2020-02-28 06:58:47 +01
categories: networking vpn
tags: wireguard mtu networking performance troubleshooting tunnel linux
---

## The Architecture of Encapsulated Traffic

WireGuard has revolutionised the VPN landscape with its simplicity and performance. However, because it operates by encapsulating IP packets within UDP, it introduces a layer of complexity that often manifests as subtle, hard-to-diagnose network issues. The most prevalent of these is the "MTU Black Hole," where small packets (like those used by Ping or SSH) pass through the tunnel without issue, but larger packets (typical of HTTP/HTTPS or file transfers) are silently dropped.

Understanding the mathematics of this encapsulation is the first step toward building a resilient tunnel that can handle any workload.

## The Mathematics of Overhead

A standard Ethernet frame has a Maximum Transmission Unit (MTU) of 1500 bytes. When a packet is sent through a WireGuard interface, it is wrapped in several headers:

1.  **UDP Header**: 8 bytes.
2.  **WireGuard Header**: 32 bytes (including the receiver index and nonce).
3.  **Authentication Tag (Poly1305)**: 16 bytes.
4.  **Outer IP Header**: 20 bytes (for IPv4) or 40 bytes (for IPv6).

For an IPv4-over-IPv4 tunnel, the total overhead is 60 bytes. However, WireGuard's default is to reserve 80 bytes to ensure compatibility with IPv6, leading to the common default MTU of **1420**.

## When the Default Fails: The Path MTU Problem

The 1420 default assumes your underlying "base" connection (the physical wire or wireless link connecting you to the internet) has a 1500-byte capacity. In many modern environments, this is not the case:

- **PPPoE (DSL/Fibre)**: Adds 8 bytes of overhead, reducing the base MTU to 1492.
- **LTE/5G and Satellite**: Often use heavy encapsulation, resulting in MTUs as low as 1380 or 1420.
- **Cloud Interconnects**: Some VPC peering or Direct Connect links have internal limits of 1450.

If your base link is 1492 (PPPoE) and your WireGuard MTU is 1420, the resulting packet is 1500 bytes (1420 + 80). The PPPoE link will drop this packet.

## The Diagnostic Phase: Path MTU Discovery (PMTUD)

Theoretically, the network should handle this automatically. When a router receives a packet that is too large for its next hop, it should send an ICMP "Destination Unreachable - Fragmentation Needed" (Type 3, Code 4) message back to the sender.

However, in many security-hardened environments, administrators block all ICMP traffic. When this happens, PMTUD fails. The sender never receives the notification and continues to send large packets that are dropped by the bottleneck router.

### Finding the Actual Path MTU

Use the `ping` utility with the "Do Not Fragment" (DF) bit to manually find the bottleneck.

```bash
# On Linux, attempt to send a 1472-byte payload (1500 total)
ping -M do -s 1472 8.8.8.8
```

Subtract 28 bytes (IP and ICMP headers) from the successful payload size to find your base MTU. If 1464 is the largest that passes, your base MTU is 1492. Your WireGuard MTU should then be set to **1412**.

## The Professional Fix: MSS Clamping

Adjusting the MTU on the `wg0` interface is a good start, but it doesn't solve issues for devices _behind_ the WireGuard gateway. The most robust solution is **MSS Clamping**. This technique intercepts the TCP Three-Way Handshake (the SYN packet) and modifies the Maximum Segment Size (MSS) value to ensure that the resulting TCP segments, after headers are added, will always fit within the tunnel MTU.

### Implementation with iptables

Add these rules to your `wg-quick` configuration to automate the clamping when the interface comes up.

```conf
[Interface]
Address = 10.10.0.1/24
PrivateKey = <KEY>
MTU = 1412

# Apply MSS Clamping for all traffic passing through the tunnel
PostUp = iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
PostDown = iptables -t mangle -D FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

{: file='/etc/wireguard/wg0.conf'}

## Advanced Performance Considerations

### 1. Fragmentation vs. CPU Load

When a packet is fragmented, the router must use its CPU to split and later reassemble the packets. This bypasses "fast-path" switching hardware. On high-speed links (1Gbps+), fragmentation can increase CPU usage by 50% or more, leading to increased jitter. Correct MTU alignment ensures "straight-through" processing.

### 2. Multi-Hop Tunnels

If you are running a VPN inside another VPN (e.g., WireGuard inside an IPsec tunnel or over a VXLAN fabric), you must account for the overhead of _both_ layers. In these complex scenarios, an MTU of **1280** is often the safest choice, as it is the minimum MTU required for IPv6.

### 3. UDP Fragmentation

WireGuard itself does not fragment. It relies on the underlying IP stack. If the encrypted UDP packets are too large for the physical path, the physical routers might fragment the UDP packets. Many firewalls drop fragmented UDP packets as a security measure. This is yet another reason why setting a conservative MTU on the `wg0` interface is critical.

## Summary of Best Practices

1.  **Measure**: Use `ping -M do` to find your actual base MTU.
2.  **Calculate**: Subtract 80 (IPv4) or 100 (IPv6) from that base MTU.
3.  **Clamp**: Always implement MSS Clamping on your WireGuard gateways to protect downstream clients.
4.  **Monitor**: Use `ip -s link show wg0` to look for errors or dropped packets which may indicate remaining MTU issues.

By following these steps, you eliminate the "intermittent freeze" issues that plague many VPN deployments, resulting in a tunnel that is as stable and performant as a physical wire. This attention to detail is what defines a professional network implementation. In 2020, WireGuard is the future of secure connectivity, but only when tuned correctly.
