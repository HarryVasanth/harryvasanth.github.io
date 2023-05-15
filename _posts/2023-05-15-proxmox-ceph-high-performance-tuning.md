---
layout: post
title: "Proxmox - Storage: High-Performance Ceph Clusters with Metadata Offloading and Network Isolation"
date: 2023-05-15 21:22:04 +01
categories: proxmox storage
tags: proxmox ceph storage ssd linux high-availability troubleshooting
---

## The Ceph Performance Paradox in Virtualisation

Ceph is widely regarded as the "Gold Standard" for distributed, self-healing storage in Proxmox clusters. However, many administrators who deploy Ceph on standard or consumer-grade hardware find themselves facing a performance nightmare: high IO Wait, sluggish VM response times, and catastrophic latency spikes during "Scrubbing" or "Recovery" events.

The reality is that Ceph is an enterprise-grade storage engine that makes specific demands on hardware and the Linux kernel. To build a cluster that delivers sub-millisecond latency, you must move beyond the default settings and implement a strategy centered on **Hardware Offloading** and **Network Isolation**.

## Core Problem: The 'Sync' Write Bottleneck

Every time a VM performs a write operation, Ceph must ensure that the data is replicated and physically committed to the stable storage of the OSDs (Object Storage Daemons).

- **The Issue**: Consumer-grade SSDs use a volatile DRAM cache to achieve high burst speeds. When Ceph issues a "Flush" command (mandatory for data integrity), the SSD must dump that cache to the slow NAND cells. This causes the SSD controller to stall for hundreds of milliseconds.
- **The Result**: Because a Ceph write is only as fast as the slowest OSD in the replication group, a single "stalling" consumer drive can bring the entire cluster to a halt.

## Strategy 1: The 'DB and WAL' Offloading (Mandatory Hardware)

The most effective way to improve Ceph performance is to separate the metadata (the database and the write-ahead log) from the actual data storage.

- **The Component**: Use a high-endurance, low-latency device like an **Intel Optane** (e.g., P1600X) or an enterprise NVMe (e.g., Samsung PM1733).
- **The Implementation**: When creating an OSD in the Proxmox UI, use the "Advanced" menu to place the **DB** and **WAL** on the fast NVMe device. A single 100GB Optane drive can handle the metadata for 5-8 standard SATA SSD OSDs. This offloads the high-frequency metadata commits, allowing the slower drives to focus purely on bulk data storage.

## Strategy 2: Network Architecture and MTU Tuning

Ceph is a network-heavy protocol. Every write is replicated at least 3 times across the network.

1.  **Dedicated Interfaces**: You **must** separate Ceph traffic from VM and Management traffic. Use dedicated NICs (ideally 10Gbps or 25Gbps) for the "Ceph Public" and "Ceph Cluster" networks.
2.  **Jumbo Frames (MTU 9000)**: Large frames reduce the CPU overhead of packet processing. This is critical when moving gigabytes of data during a cluster rebalance.
    - _Note_: Ensure that every switch and NIC in the path supports MTU 9000, or you will experience intermittent connection failures.

## Strategy 3: Kernel and OSD Software Tuning

To further refine performance, you can adjust how Ceph interacts with the Linux kernel:

### 1. Reducing Scrubbing Impact

"Scrubbing" is the process where Ceph checks for data corruption. By default, it can be very aggressive.

```bash
# Reduce the priority of scrubbing to avoid impacting VM IOPS
ceph config set osd osd_scrub_priority 1
ceph config set osd osd_scrub_sleep 0.1
```

### 2. Tuning the I/O Scheduler

For SSD-based OSDs, the Linux kernel's default I/O scheduler (often `mq-deadline` or `kyber`) can add unnecessary overhead.

```bash
# Set the scheduler to 'none' for all OSD disks
echo none > /sys/block/sdX/queue/scheduler
```

### 3. Increasing File Descriptors

A Ceph OSD handles thousands of connections. Ensure the system limits are high enough.

```text
# /etc/security/limits.conf
* hard nofile 1048576
* soft nofile 1048576
```

## Monitoring: The Advanced Diagnostic Toolkit

You cannot tune what you cannot see. Use these commands to identify bottlenecks:

- **`ceph -s`**: The high-level health overview.
- **`ceph osd perf`**: Identifies which specific OSD is suffering from high commit latency.
- **`ceph osd df tree`**: Shows the distribution of data across your disks.
- **`iostat -xz 1`**: Monitors the %Util and await metrics for each physical drive.

## Summary

A high-performance Ceph cluster is the result of intentional architectural choices. By offloading metadata to low-latency flash, isolating replication traffic on a 10Gbps+ network, and tuning the OSD parameters to respect VM priority, you transform Ceph from a "slow and heavy" system into a "fast and resilient" enterprise storage backbone. This level of rigour is what defines a highly skilled Proxmox administrator and ensures a stable platform for mission-critical workloads. Always remember: in Ceph, latency is the only metric that truly matters for user experience.
