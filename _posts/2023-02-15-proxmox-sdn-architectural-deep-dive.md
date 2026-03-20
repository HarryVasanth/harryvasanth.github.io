---
layout: post
title: "Proxmox - Networking: Architectural Deep Dive into SDN, VxLAN, and EVPN Control Planes"
date: 2023-02-15 02:34:15 +01
categories: proxmox networking
tags: proxmox sdn vxlan evpn networking virtualisation administration
---

## The Transition from Static Bridges to Dynamic Fabrics

In the early days of Proxmox virtualisation, the standard approach to networking was the Linux Bridge (`vmbr0`). While robust, it tied the virtual machine to a specific physical broadcast domain. If you wanted to move a VM to a different subnet or stretch a network across multiple locations, you were at the mercy of physical switch configurations and the 4096-ID limit of 802.1Q VLANs.

**Software-Defined Networking (SDN)** in Proxmox (v7.2+) represents a paradigm shift. It decouples the logical network from the physical cables. By using encapsulation protocols like **VxLAN** and control planes like **EVPN**, we can build a high-performance network fabric that scales to millions of isolated segments, all while remaining completely transparent to the physical infrastructure.

## Core Pillars of the Proxmox SDN Stack

To build a production-ready network, you must master the three logical layers of the SDN stack:

### 1. The Zone (The Encapsulation Technology)

The Zone is the definition of _how_ packets are moved between nodes.

- **Simple**: For local isolation on a single node (using NAT or Bridges).
- **VLAN**: For standard 802.1Q tagging across a cluster.
- **VxLAN**: The modern choice. It wraps Layer 2 Ethernet frames inside Layer 3 UDP packets (Port 4789). This allows your VMs to "see" each other across different subnets as if they were on the same physical switch.
- **EVPN**: The ultimate control plane. It uses **BGP** to explicitly share MAC and IP address locations between all Proxmox nodes, eliminating the need for inefficient multicast "Flood and Learn."

### 2. The VNet (The Virtual Bridge)

The VNet is the object that the VM actually "plugs into." In the VM hardware settings, you select a VNet instead of a bridge. Because VNets are cluster-wide, when a VM migrates, its networking follows it automatically without any configuration changes.

### 3. The Subnet (Integrated IPAM)

Proxmox SDN includes a built-in IP Address Management (IPAM) system. You can define subnets for your VNets, and Proxmox will automatically assign IPs to VMs, track usage, and even update internal DNS records.

## Implementation: Building a Resilient EVPN Fabric

### Step 1: Preparing the Infrastructure

Before enabling SDN, ensure that **FRRouting (FRR)** is installed on all cluster nodes. This daemon handles the BGP sessions that power the EVPN control plane.

```bash
apt update && apt install frr-pythontools pve-network-router-repository
```

### Step 2: Defining the BGP Controller

In the Proxmox Datacenter panel, create a new **BGP Controller**.

- **ASN**: Use a private Autonomous System Number (e.g., `65001`).
- **Peers**: List the IP addresses of every node in your cluster. This establishes the "Full Mesh" peering required for EVPN.

### Step 3: Creating the EVPN Zone

Create an **EVPN Zone** and link it to your BGP Controller.

- **MTU**: Set this to **1450** (to account for the 50-byte VxLAN overhead on standard 1500-byte networks).
- **VNet MAC**: Use an Anycast MAC address. This allows the Proxmox host to act as the default gateway for its VMs, providing ultra-low latency "First Hop" routing.

## The Critical 'MTU' and 'Jumbo Frames' Strategy

The most common failure in SDN deployments is the "TCP Stall." Small packets (Ping, DNS) work, but large packets (HTTPS, File Transfer) hang. This is due to the VxLAN header pushing packets over the standard 1500-byte limit.

**The Ideal Fix**:

1.  **Underlay Tuning**: Enable Jumbo Frames (MTU 9000) on your physical switches and Proxmox physical NICs.
2.  **Overlay Alignment**: Ensure the VMs themselves are configured for an MTU of 1450 if you cannot use Jumbo Frames. This prevents the silent fragmentation that kills performance.
3.  **MSS Clamping**: If you have traffic leaving the SDN to the internet, implement MSS clamping on your perimeter firewall.

## Advanced: Monitoring the EVPN Control Plane

Because EVPN relies on BGP, you can use standard networking tools to debug the state of your fabric.

- **Check the BGP Peering**:

```bash
vtysh -c "show bgp summary"
```

- **Verify MAC Learning**:

```bash
vtysh -c "show bgp evpn route"
```

If you don't see the MAC addresses of your VMs in the BGP table, the encapsulation will fail, and the packets will be dropped at the ingress node.

## Why This Architecture Wins for 2023

1.  **Workload Mobility**: VMs can move between physical datacenters as long as there is L3 connectivity between Proxmox hosts.
2.  **Multi-Tenancy**: You can provide identical subnets (e.g., `10.0.0.0/24`) to different clients without them ever overlapping or seeing each other's traffic.
3.  **Reduced Blast Radius**: A broadcast storm on one VNet is confined to that segment and does not impact the rest of the cluster or the physical switches.

## Summary

Proxmox SDN is not just a feature; it is the foundation for a modern, automated datacenter. By mastering EVPN-VxLAN, you provide your organisation with the agility of a public cloud within your own infrastructure. You move from "Managing Switches" to "Architecting a Fabric," ensuring that your network is as flexible and resilient as the virtual workloads it supports. This is the definitive standard for highly skilled network and virtualisation administrators in 2023.
