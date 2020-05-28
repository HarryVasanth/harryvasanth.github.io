---
layout: post
title: "Proxmox - ZFS: Eliminating IO Wait with ARC Tuning and SLOG Optimisation"
date: 2020-05-28 21:00:28 +01
categories: proxmox storage
tags: proxmox zfs ssd performance storage linux administration tuning
---

## The ZFS RAM Paradox in Hypervisors

Proxmox administrators often encounter a confusing situation: their host server reports 95% RAM usage even when only a few small Virtual Machines are running. In almost every case, the culprit is the ZFS **Adaptive Replacement Cache (ARC)**. While ZFS is designed to intelligently release this memory when applications demand it, the Linux kernel's memory pressure algorithm is not always fast enough to satisfy the immediate, large memory requests of a starting KVM process. This leads to system stuttering, high IO Wait, and in extreme cases, the OOM (Out of Memory) killer terminating your VMs.

To build a professional-grade hypervisor that remains responsive under heavy load, you must move beyond the default settings and manually tune the interaction between ZFS and the kernel.

## Tuning Strategy 1: Constraining the ARC

In a dedicated storage server, you want ZFS to use as much RAM as possible for caching. In a hypervisor, however, the RAM belongs to the VMs. You must establish a strict boundary for ZFS. A proven rule of thumb for Proxmox hosts is to allocate 1GB of ARC for every 1TB of physical storage, with a minimum of 4GB and a maximum of 20% of your total system RAM.

### Implementation Procedure

To set a hard limit of 8GB for the ARC, edit the ZFS module configuration:

```bash
# Create or edit the configuration file
sudo nano /etc/modprobe.d/zfs.conf

# Add the following line (value is in bytes: 8 * 1024 * 1024 * 1024)
options zfs zfs_arc_max=8589934592

# Apply the changes to the initramfs boot image
update-initramfs -u
reboot
```

After rebooting, you can verify the current ARC size using `arcstat` or `arc_summary`.

## Tuning Strategy 2: Physical Sector Alignment (Ashift)

Modern SSDs and NVMe drives use 4KB physical sectors, even if they report 512B to the OS for compatibility. If your ZFS pool was created with the default `ashift=9` (representing 512B), ZFS will perform many small writes that don't align with the hardware's internal pages. This triggers a "read-modify-write" cycle inside the SSD controller, which drastically reduces performance and leads to premature disk failure due to write amplification.

**Verification**:

```bash
zdb -C | grep ashift
```

If the output is not `12`, your pool is misaligned. Unfortunately, `ashift` cannot be changed after a pool is created. To fix this, you must back up your data, destroy the pool, and recreate it using the `-o ashift=12` flag. This is a "Day 0" task that a experienced administrator always verifies before putting a server into production.

## Tuning Strategy 3: The ZIL and Dedicated Log Devices (SLOG)

When a VM performs a "Sync" write (common in database operations), ZFS must ensure the data is physically on stable storage before acknowledging the write to the guest. It records these writes in the ZFS Intent Log (ZIL). On an all-SSD pool, the ZIL shares the same NAND cells as your data, creating contention and increasing latency.

If you have database-heavy workloads, adding a dedicated **SLOG** (Separate Log) using a high-endurance, low-latency device like an **Intel Optane** can significantly reduce write latency. Because the SLOG only handles writes that haven't been committed to the main pool yet, it only needs to be a few gigabytes in size.

```bash
# Add a small mirrored partition on two Optane drives as SLOG
zpool add <pool_name> log mirror /dev/nvme0n1p1 /dev/nvme1n1p1
```

## Tuning Strategy 4: Transaction Group (TXG) Timeout

ZFS collects writes in memory and flushes them to the physical disks in a single large batch every 5 seconds. This is the Transaction Group sync. On systems with many active VMs, this 5-second pulse can cause a periodic "hiccup" where the whole system feels sluggish for a few hundred milliseconds every five seconds.

You can create a smoother I/O flow by making the flushes more frequent but smaller:

```bash
# Set the sync timeout to 1 second instead of 5
echo 1 > /sys/module/zfs/parameters/zfs_txg_timeout
```

To make this persistent, add `options zfs zfs_txg_timeout=1` to your `/etc/modprobe.d/zfs.conf`.

## Summary: The SysAdmin's Checklist

By constraining the ARC, ensuring correct sector alignment, and offloading synchronous writes to high-speed hardware, you transform ZFS from a memory-hungry monolith into a precision-tuned storage engine. This level of optimisation is what allows a Proxmox host to support hundreds of high-performance virtual machines with the stability required for enterprise production environments. Always monitor your IO Wait and ARC hit rates to ensure your tuning matches your actual workload. In 2020, ZFS is the foundation of the modern datacenter storage stack.
