---
layout: post
title: "Proxmox - Networking: Deep Dive into EVPN-VXLAN Control Planes and Multi-Tenant Isolation"
date: 2024-02-15 04:59:51 +01
categories: proxmox networking
tags: proxmox sdn evpn vxlan networking virtualisation multi-site administration
---

## The Evolution of the Virtual Network Fabric

In modern datacenter architecture, we are moving away from simple VLANs and even simple VXLAN overlays. The challenge with a standard VXLAN setup is the management of the "Control Plane." How does Node A know that a specific MAC address has moved to Node B? In a basic setup, we rely on "Flood and Learn" (multicast), which is inefficient and doesn't scale well across different subnets or sites.

**EVPN (Ethernet VPN)** provides the sophisticated control plane we need. Using BGP as the transport, EVPN allows Proxmox nodes to explicitly share MAC and IP address information. This turns your cluster into a high-performance, programmable network fabric that can span multiple physical locations with zero packet loss during VM migration.

## Core Concepts of EVPN in Proxmox

To build a professional EVPN fabric, you must understand three components:

1.  **VTEP (VXLAN Tunnel End Point)**: The Proxmox host that performs the encapsulation.
2.  **BGP Control Plane**: Used to distribute routing information (Layer 2 MACs and Layer 3 IPs) between VTEPs.
3.  **Anycast Gateway**: Allows a VM to use the same default gateway IP regardless of which physical node it is running on.

## Implementation: Building an EVPN Zone

### Step 1: Prerequisites: Installing the FRR Plugin

Proxmox uses **FRRouting (FRR)** to handle the BGP/EVPN logic. This must be present on all nodes.

```bash
apt update && apt install frr-pythontools pve-network-router-repository
```

### Step 2: Configure the 'BGP Controller'

In the Proxmox UI (Datacenter -> SDN -> Controllers):

- **ID**: `bgp-controller`
- **Type**: `bgp`
- **ASN**: Use a private ASN (e.g., `65000`).
- **Peers**: Enter the IPs of your other Proxmox nodes. This creates a "Full Mesh" BGP peering for internal route distribution.

### Step 3: Define the EVPN Zone

Navigate to Datacenter -> SDN -> Zones:

- **ID**: `evpn-zone`
- **Type**: `evpn`
- **Controller**: `bgp-controller`
- **VNet MAC Address**: (Optional) Use an Anycast MAC if you want seamless L3 routing across nodes.
- **Advertise Subnets**: Enable this to allow the fabric to route between different VNets without an external router.

### Step 4: Create the VNets

Now you can create logical VNets attached to the `evpn-zone`. These VNets function like distributed switches that span every node in your cluster.

## Architecture: Why EVPN is the Ideal Choice

- **Optimal Forwarding**: Traffic takes the shortest path between nodes. If two VMs are on the same node, traffic never leaves the physical box.
- **Multi-Homing Support**: EVPN supports active-active links to the physical network, providing massive bandwidth and redundancy.
- **Scalability**: EVPN is the standard used by the world's largest cloud providers. By implementing it in Proxmox, you are future-proofing your infrastructure for hundreds of nodes and thousands of isolated networks.

## The MTU Trap (Revisited)

Just like standard VXLAN, EVPN/VXLAN adds overhead (50 bytes).

- **The Advanced Rule**: You **must** ensure the underlying physical network supports Jumbo Frames (9000 bytes) or carefully tune the internal MTU to **1450**. Failure to do this will result in mysterious packet drops and "TCP Stalls" that are notoriously difficult to debug.
- **Verification**: Use `ping -M do -s 1422` to verify that the tunnel can carry the payload without fragmentation.

## SDN Troubleshooting Commands

Proxmox provides a set of CLI tools for the SDN layer:

```bash
# Check the status of the SDN zones
pvesh get /cluster/sdn/zones

# Check the generated FRR configuration
cat /etc/frr/frr.conf

# View the BGP EVPN routing table
vtysh -c "show bgp evpn route"
```

## Summary: A Cloud-Scale Network in Your Basement

Proxmox SDN with EVPN represents the pinnacle of virtual networking. It removes the physical network as a constraint, allowing your infrastructure to grow and move as your workloads require. By leveraging BGP for the control plane, you gain a level of reliability and visibility that is impossible with traditional Layer 2 networking. This is the definitive architecture for any experienced administrator managing a multi-node, multi-tenant environment. It turns your cluster into a single, massive distributed switch.
