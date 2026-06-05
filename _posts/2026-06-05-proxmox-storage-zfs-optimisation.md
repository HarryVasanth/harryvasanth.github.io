---
layout: post
title: "Proxmox - Storage: Advanced ZFS Optimisation for High-Density NVMe Clusters"
date: 2026-06-05 18:42:02 +01
categories: proxmox storage
tags: proxmox zfs linux performance nvme virtualisation administration
---

## The 'Default Configuration' Bottleneck

Out of the box, ZFS is a brilliant filesystem. It provides mathematical data integrity, instant snapshots, and seamless replication. However, the default ZFS configuration provided by Proxmox is designed to be completely safe for a standard hard drive array. When you deploy a cluster of hypervisors backed by enterprise NVMe storage in 2026, those default settings become a strict bottleneck.

A experienced infrastructure administrator understands that hardware is only as fast as the kernel module managing it. If you are running dozens of database Virtual Machines (VMs) on a single host, default ZFS settings will cause write amplification, excessive memory swapping, and artificial latency spikes. This guide explores the critical ZFS tuning parameters required to unlock bare-metal performance for high-density virtualisation workloads.

## Phase 1: Taming the ARC (Adaptive Replacement Cache)

ZFS uses the ARC to store frequently accessed data directly in system RAM. By default, ZFS will consume up to 50% of your host memory for this cache. In a storage appliance, this is perfect. In a hypervisor environment where your VMs need that RAM, this causes massive memory pressure. The Linux kernel will frequently fight with ZFS over memory allocation, triggering the Out-Of-Memory (OOM) killer.

You must explicitly restrict the maximum ARC size to leave enough breathing room for your virtual machines.

### Implementation: Restricting ARC Size

We configure this using a kernel module parameter. Let us assume we have a host with 256GB of RAM, and we want to restrict ZFS to use a maximum of 16GB.

```bash
# Calculate 16GB in bytes: 16 * 1024 * 1024 * 1024 = 17179869184
echo "options zfs zfs_arc_max=17179869184" > /etc/modprobe.d/zfs.conf

```

To apply this without a reboot, you can dynamically update the parameter via `sysfs`:

```bash
echo 17179869184 > /sys/module/zfs/parameters/zfs_arc_max

```

## Phase 2: Combating Write Amplification (volblocksize)

When you create a VM disk in Proxmox on ZFS, it creates a "zvol" (a raw block device). The default block size (`volblocksize`) for a zvol is 8K.

If you install a modern Linux VM using the ext4 or XFS filesystem, that guest OS writes data in 4K blocks. This creates a severe mismatch. Every time the guest VM writes a 4K block, ZFS has to read the underlying 8K block, modify half of it, and write the full 8K back to the disk. This is known as "Write Amplification". It halves your NVMe lifespan and destroys random write performance.

### The Optimal Solution: Application-Aware Alignment

You must align your `volblocksize` to match the workload inside the VM.

- **General Linux/Windows VMs**: 16K or 32K provides a great balance.
- **Database VMs (PostgreSQL/MySQL)**: 8K or 16K depending on the specific database page size.

You cannot change the `volblocksize` of an existing disk. You must set it before creating the VM or define it globally for the storage pool in Proxmox.

To manually create a highly optimised database zvol:

```bash
zfs create -V 100G -o volblocksize=16K rpool/data/vm-101-disk-0

```

## Phase 3: The 'Sync' Parameter and Enterprise NVMe

By default, ZFS is heavily focused on data safety. When a database VM requests a synchronous write, ZFS writes it to a special staging area called the ZIL (ZFS Intent Log) before committing it to the main pool. If your ZIL is located on the same drives as your main storage, you are effectively writing every piece of data twice.

If you are using Enterprise NVMe drives (like the Intel Optane or modern Samsung PM-series) which have built-in Power Loss Protection (PLP), this double-write is unnecessary overhead. The drive's internal capacitors guarantee that data in transit will survive a sudden power failure.

### Implementation: Selectively Disabling Sync

**Warning**: Do not do this on consumer-grade SSDs, or you risk total pool corruption during a power outage.

If your hardware supports PLP, or if you are running a strictly clustered database (like a 3-node Galera cluster) that handles its own replication, you can safely disable synchronous writes for that specific dataset to drastically reduce latency.

```bash
# Disable sync for a specific virtual machine disk
zfs set sync=disabled rpool/data/vm-101-disk-0

```

## Phase 4: Compression and CPU Overhead

ZFS supports transparent compression. The default algorithm is `lz4`, which is incredibly fast and aborts early if data is uncompressible. In 2026, CPUs are so powerful that the time taken to compress the data is actually less than the time it takes to write uncompressed data to the physical NVMe bus.

However, the newer `zstd` algorithm offers a much better compression ratio. The secret to high-performance virtualisation is using `zstd` with a low compression level.

```bash
# Set fast, modern compression on the entire VM pool
zfs set compression=zstd-1 rpool/data

```

This reduces the physical footprint of your VMs by 30 to 40 percent with virtually zero CPU penalty, effectively giving you free storage and reducing the I/O load on your drives.

## Summary

High-density virtualisation is a balancing act between the hypervisor, the storage layer, and the guest operating system. By explicitly managing the ARC, aligning your block sizes, leveraging hardware PLP, and upgrading your compression algorithms, you transform a generic Proxmox installation into an enterprise-grade storage appliance.

This level of granular tuning is the difference between an infrastructure that simply runs, and an infrastructure that performs flawlessly under intense, real-world pressure.
