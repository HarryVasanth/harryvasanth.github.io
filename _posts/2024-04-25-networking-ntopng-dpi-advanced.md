---
layout: post
title: "Networking - Monitoring: Deep Packet Inspection and Flow Analysis with ntopng and ClickHouse"
date: 2024-04-10 13:26:32 +01
categories: networking monitoring
tags: ntopng nprobe networking dpi monitoring troubleshooting linux administration
---

## Beyond the 'NetFlow' Aggregate

Standard network monitoring (like SNMP or basic NetFlow) tells you _how much_ traffic is flowing, but it rarely tells you _what_ that traffic actually is. You might see a 1Gbps spike on your WAN interface, but is it a legitimate backup job, a user watching 4K YouTube, or a data exfiltration event?

To answer these questions, you need **Deep Packet Inspection (DPI)**. **ntopng** is the gold standard for open-source high-speed network analysis. It goes beyond simple headers and looks at the actual payload to identify over 300 different application protocols (using the `nDPI` library).

## The Architecture of High-Speed Analysis

For a professional deployment, you shouldn't run ntopng directly on your core router (it's too CPU-intensive). Instead, use a **Port Mirror (SPAN)** or a **Network TAP** to send a copy of all traffic to a dedicated Linux monitoring server.

1.  **The Source**: Your core switch or OPNsense/Mikrotik router.
2.  **The Collector (nProbe)**: A lightweight daemon that receives the raw packets, performs the DPI analysis, and generates "Flows."
3.  **The Engine (ntopng)**: Receives the flows from nProbe, stores them in a database (like ClickHouse), and provides the web-based analytics.

## Implementation: Setting up the Mirror and Collector

### Step 1: Prepare the Linux Interface

Ensure the monitoring interface is in "Promiscuous Mode" and doesn't have an IP address (to prevent the server from trying to talk on the mirror port).

```bash
ip link set eth1 up
ip link set eth1 promisc on
```

### Step 2: Start nProbe

We tell nProbe to listen on the physical interface and send processed flows to the ntopng instance via ZMQ.

```bash
nprobe --zmq "tcp://127.0.0.1:5556" -i eth1 -n none
```

### Step 3: Configure ntopng

ntopng listens for the flows from nProbe and provides the visualization.

```bash
# /etc/ntopng/ntopng.conf
-i="tcp://127.0.0.1:5556"
-w=3000
-n=1
```

## Advanced Use Case: Identifying "Shadow IT" and Security Risks

ntopng's power lies in its ability to categorize traffic into "Host Pools" and "Application Categories."

- **Lateral Movement**: If a printer (Host Pool: IoT) suddenly starts a SSH session (Category: Remote Access) to your Database Server, ntopng will trigger an immediate alert.
- **DNS Exfiltration**: ntopng identifies DNS queries that are unusually large or sent to non-standard servers, a common sign of tunneling or malware command-and-control.
- **SaaS Usage**: You can instantly see how much bandwidth is going to Office 365 vs. Google Workspace vs. unauthorized Dropbox accounts.

## Performance Tip: The ClickHouse Backend

By default, ntopng stores historical data in RRD files, which are limited. For a production-ready deployment, you **must** configure ntopng to export data to **ClickHouse**. ClickHouse is a columnar database that can handle billions of network flows and allow you to perform sub-second SQL queries.

```bash
# Example SQL to find the top 5 protocols by volume in the last hour
SELECT protocol, sum(bytes) FROM flows
WHERE timestamp > now() - 3600
GROUP BY protocol
ORDER BY sum(bytes) DESC LIMIT 5;
```

## Summary

Network visibility is the foundation of both security and performance. By implementing a DPI-aware stack with ntopng and nProbe, you move from "guessing" to "knowing." You gain the mathematical proof needed to justify bandwidth upgrades, identify compromised hosts, and enforce corporate usage policies. It is an essential component of a modern, "Zero Trust" infrastructure strategy. It turns the network from a "Black Box" into a "Transparent Stream of Intelligence."
