---
layout: post
title: "Networking - Linux Bridges: Eliminating the 30-Second 'Silence' with STP Tuning"
date: 2021-11-05 18:45:12 +01
categories: networking linux
tags: stp networking bridges troubleshooting performance linux
---

## The Problem: The Missing Cloud-init Data

You've automated your VM deployments, but you notice a strange issue: the VMs boot, but they fail to fetch their configuration from the network. If you wait 30 seconds and reboot, they work perfectly.

This is because the Linux Bridge, by default, runs the **Spanning Tree Protocol (STP)**. When an interface (a VM's tap device) is added to the bridge, the bridge waits for the "Listening" and "Learning" states to complete before it starts forwarding packets. This takes exactly 30 seconds.

## The Optimal Solution: Edge Port Configuration

If you are certain that your bridge is not connected to another switch in a way that could create a loop, you should reduce the **Forward Delay**.

### 1. The Netplan Implementation (Ubuntu/Debian)

```yaml
network:
  version: 2
  bridges:
    br0:
      interfaces: [enp0s3]
      parameters:
        stp: true
        forward-delay: 2 # Reduce from 15 (per state) to 1 or 2
```

{: file='/etc/netplan/01-netcfg.yaml'}

### 2. The CLI (Immediate) Fix

```bash
sudo brctl setfd br0 2
```

## Why not just disable STP?

Disabling STP entirely (`stp: false`) is an option, but it's dangerous. If someone accidentally connects two bridge-linked interfaces together, you will create a broadcast storm that can take down your entire host and the physical switch it's connected to. Setting a low `forward-delay` is the optimal choice: it provides the speed needed for PXE/Cloud-init while still offering a safety net against loops.

## Tips & Tricks

- **VLAN Filtering**: If you are using a single bridge for multiple VLANs, enable `vlan-filtering`. This allows you to treat the bridge like a managed switch:
  ```bash
  ip link set br0 type bridge vlan_filtering 1
  ```
- **MAC Learning**: On high-density hosts with thousands of containers/VMs, the bridge's MAC table can become a bottleneck. You can increase the `ageing-time` or use static FDB entries for known VMs to reduce CPU load.
- **The 'Hairpin' Mode**: If two VMs on the same bridge need to communicate via an external gateway, you may need to enable `hairpin` mode on the bridge ports.

## Summary

Network automation is only as good as the underlying plumbing. By tuning your Linux bridges for the specific demands of virtualisation, you ensure that your automated provisioning flows are fast, reliable, and "just work" the first time.
