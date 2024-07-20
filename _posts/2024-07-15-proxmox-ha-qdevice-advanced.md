---
layout: post
title: "Proxmox - HA: Mastering Cluster Quorum with QDevice and External Arbiters"
date: 2024-07-20 16:31:46 +01
categories: proxmox high-availability
tags: proxmox ha qdevice cluster networking troubleshooting administration
---

## The Two-Node Cluster Trap

A fundamental rule of Proxmox High Availability (HA) is the **Principle of Quorum**. To make decisions (like fencing a failed node or migrating a VM), the cluster must have a majority of votes (> 50%).

- In a 3-node cluster, if one node fails, 2 nodes remain (66%). Quorum is maintained.
- In a 2-node cluster, if one node fails, only 1 node remains (50%). **Quorum is lost.**

When quorum is lost, the remaining node goes into "Read-Only" mode. It will not start any VMs, and it cannot perform HA failover. This makes a 2-node HA cluster essentially useless during a real failure.

The ideal solution is the **QDevice (Quorum Device)**. This allows a third, lightweight device (like a Raspberry Pi or a tiny cloud VPS) to provide the tie-breaking vote without needing to be a full member of the Proxmox cluster.

## How QDevice Works: The Corosync-QNet Mechanism

The QDevice runs the `corosync-qnetd` daemon. It doesn't store VM data, and it doesn't need high CPU or RAM. It simply participates in the Corosync protocol and provides a vote to whichever "side" of a network partition has the most nodes. This allows a 2-node cluster to survive a single node failure because the remaining node (1 vote) + the QDevice (1 vote) = 2 out of 3 votes (66%).

## Implementation: Adding a QDevice to your Cluster

### Step 1: Prepare the External Device (The Arbiter)

On a separate Debian/Ubuntu server (not part of the cluster):

```bash
sudo apt update && sudo apt install corosync-qnetd
```

### Step 2: Prepare the Proxmox Nodes

On **all** nodes in your Proxmox cluster:

```bash
apt update && apt install corosync-qdevice
```

### Step 3: Link the Cluster to the QDevice

From **one** of your Proxmox nodes, run the setup command. You will need SSH access to the arbiter.

```bash
pvecm qdevice setup <IP_OF_ARBITER>
```

Proxmox will automatically handle the certificate exchange and update the `/etc/pve/corosync.conf` across the whole cluster.

## Verification: Checking the Vote Distribution

```bash
pvecm status
```

Look for the `Quorum information` section. You should see:

- `Votes: 3` (Node A + Node B + QDevice)
- `Quorum: 2` (The minimum needed to make decisions)
- `Device: Yes (Type: qdevice)`

## Advanced Use Case: The 'Split-Brain' Protector

In a "Split-Brain" scenario, the network link between Node A and Node B is broken, but both nodes are still running.

- **Without QDevice**: Both lose quorum. All VMs stop.
- **With QDevice**: The QDevice can only talk to one of the nodes (whichever one it still has a link to). It gives its vote to that node. That node maintains quorum and keeps the VMs running. The other node, seeing it is in the minority, gracefully fences itself (reboots) to prevent data corruption.

## Architecture Tip: Placement and Latency

The QDevice **must** be on a separate power circuit and a separate network switch from your main cluster. If the QDevice fails at the same time as one of your nodes, you are back to having no quorum.

- **Latency**: Corosync is sensitive to latency. Ideally, the QDevice should be within 10-20ms of the main cluster. A small VPS in a nearby cloud region is an excellent choice.

## Summary

High Availability is as much about mathematics as it is about hardware. By implementing a QDevice, you transform a fragile 2-node setup into a resilient, production-grade cluster. It is the most cost-effective way to achieve true HA for small-scale infrastructure deployments, providing the peace of mind that a single failure won't bring down your entire environment. It represents the move from "Physical Nodes" to "Logical Quorum Mastery."
