---
layout: post
title: "Networking - BGP: Scaling with the 'BIRD' Internet Routing Daemon"
date: 2025-06-25 07:26:04 +01
categories: networking linux
tags: bgp bird networking linux routing administration
---

## The Transition from 'Static' to 'Dynamic' Infrastructure

As your infrastructure grows from a few servers to multiple racks or datacenters, static routing becomes an operational bottleneck. You cannot manually update routes every time a new subnet is added or a provider link fails. You need a protocol that handles the "Intelligence" of the network. While we have discussed OSPF for internal use, **BGP (Border Gateway Protocol)** is the standard for connecting to ISPs and cloud providers.

**BIRD** is the leading Linux routing daemon for high-performance BGP. It is used by major Internet Exchange Points (IXPs) and CDNs because of its incredible speed, low memory footprint, and powerful configuration language.

## Architecture: The BIRD Routing Table

Unlike simpler daemons, BIRD can manage multiple independent routing tables. It uses "Protocols" (which are actually internal modules) to move routes between these tables.

1.  **Protocol Kernel**: Moves routes between BIRD and the Linux Kernel's routing table.
2.  **Protocol Static**: Manages routes you want to define manually.
3.  **Protocol BGP**: Communicates with external routers.

## Implementation: Peering with an Upstream ISP

Imagine you have been assigned an AS (Autonomous System) number `65001` and a prefix `192.0.2.0/24`.

```text
# /etc/bird/bird.conf
router id 10.0.0.1;

# Synchronise BIRD with the OS
protocol kernel {
    ipv4 {
        export all;  # Send BIRD routes to the Linux kernel
        import all;  # Take Linux routes into BIRD
    };
    persist;
}

# Define our own prefix
protocol static my_networks {
    ipv4;
    route 192.0.2.0/24 reject; # Internal 'blackhole' for the whole range
}

# The BGP Session
protocol bgp upstream_isp {
    local as 65001;
    neighbor 10.0.0.2 as 64999;
    ipv4 {
        # ONLY export our official prefix to the ISP
        export filter {
            if net = 192.0.2.0/24 then accept;
            reject;
        };
        import all; # Take all routes from the ISP (the Internet)
    };
}
```

## Suitable Strategy: The 'Filter' is the Key

The power of BIRD lies in its C-like filtering language. You can write complex logic to decide which routes to trust and which to ignore.

```text
filter secure_import {
    # Drop "Martians" (private IPs that shouldn't be on the internet)
    if net ~ [ 10.0.0.0/8+, 172.16.0.0/12+, 192.168.0.0/16+ ] then reject;

    # Drop routes with an unnaturally long AS path (prevents leaks)
    if bgp_path.len > 10 then reject;

    accept;
}
```

## Performance: BIRD vs. the World

BIRD is written in highly optimised C. It can handle a "Full Internet Table" (over 900,000 routes) using less than 200MB of RAM. This makes it possible to build a multi-terabit router using a standard, low-cost Linux server and a few high-speed NICs.

## Summary

BGP is the language of the internet, and BIRD is its most eloquent speaker. By adopting BIRD for your core routing, you gain the ability to scale your network to any size, multi-home with different ISPs for 100% uptime, and implement sophisticated traffic engineering. It is the definitive tool for the experienced network administrator moving beyond the limits of traditional LANs.
