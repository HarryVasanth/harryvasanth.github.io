---
layout: post
title: "Networking - iperf3: Advanced Throughput Testing and BDP Calculations"
date: 2022-03-11 19:00:58 +01
categories: networking performance
tags: iperf3 networking performance-tuning bdp linux
---

## The Problem: The 10Gbps Link that only does 2Gbps

You've just installed a shiny 10Gbps fibre link between two datacenters. You run `iperf3 -c <remote_ip>` and see a disappointing 2.1 Gbps. You check CPU and disk I/O, and both are idle. Why is the network underperforming?

## The Root Cause: BDP and TCP Window Scaling

TCP is a "reliable" protocol; it waits for an acknowledgement (ACK) before sending more data. The amount of data that can be "in flight" is limited by the **TCP Window Size**.

**BDP (Bandwidth-Delay Product)** is the formula: `Bandwidth (bits/sec) * Round Trip Time (seconds)`.
If your RTT is 20ms and your link is 10Gbps:
`10,000,000,000 * 0.020 = 200,000,000 bits (approx 25MB)`.

If your Linux TCP window is capped at 2MB (the default on many older distros), you will never exceed ~800Mbps on that link, regardless of physical capacity.

## High-level iperf3 Methodology

### 1. Identify the Bottleneck with Parallel Streams

If one CPU core is saturated by a single TCP stack thread, use multiple streams to distribute the load.

```bash
iperf3 -c 10.0.0.1 -P 8
```

If the total throughput jumps to 9Gbps, your problem is per-stream CPU or single-stream window limits.

### 2. Testing the 'Raw' Capacity (UDP)

UDP doesn't care about ACKs or windows. Use it to see what the physical wire can actually handle.

```bash
iperf3 -c 10.0.0.1 -u -b 10G
```

Look for **Jitter** and **Packet Loss**. If loss is >1%, you have a physical layer issue or a congested switch buffer.

### 3. Testing Reverse Path

Often, ISP or firewall rules differ for upload vs download.

```bash
iperf3 -c 10.0.0.1 -R
```

## Tips & Tricks

- **MSS Alignment**: Use the `-M` flag to test different MTU/MSS sizes. If throughput drops significantly when you go from 1460 to 1470, you likely have a Path MTU Discovery (PMTUD) failure.
- **JSON Output for Automation**: Use `--json` to pipe results into a script for periodic baseline testing. A well-versed admin always has a "baseline" to compare against when a user complains the network is "slow."
- **Zero-Copy**: Use the `-Z` flag to use the `sendfile()` system call, which reduces CPU overhead by avoiding copying data between kernel and user space.

## Summary

Don't trust a single `iperf3` run. By understanding BDP and using parallel streams, reverse testing, and UDP baselines, you can definitively prove where a network bottleneck exists-be it in the hardware, the OS tuning, or the application.
