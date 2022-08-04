---
layout: post
title: "Linux - NFS: Tuning nfsd Threads and TCP Windows for 10Gbps+ Storage"
date: 2022-08-05 00:40:22 +01
categories: linux storage
tags: nfs performance storage optimisation networking linux
---

## The Problem: The 100MB/s Ceiling

You have a high-speed 10Gbps network and a fast SSD-backed storage server. Yet, when mounting via NFS, your `rsync` or database backups are capped at around 110-120MB/s (the limit of a 1Gbps link). You check `iperf3`, and the network is fine. The bottleneck is the NFS server configuration.

## Advanced Solution 1: nfsd Thread Count

By default, most Linux distributions start the NFS server with only **8 threads**. Each thread can handle one I/O request at a time. In a multi-client environment or even with a single high-concurrency client, these 8 threads become saturated instantly.

### Implementation

Edit `/etc/default/nfs-kernel-server` (Debian/Ubuntu) or `/etc/sysconfig/nfs` (RHEL):

```text
# Increase from 8 to 128 or 256
RPCNFSDCOUNT=128
```

Restart the service: `sudo systemctl restart nfs-kernel-server`.

## Advanced Solution 2: Buffer Sizes (rsize/wsize)

The default `rsize` and `wsize` (read/write block sizes) are often 32KB or 64KB. For 10Gbps networks, this results in massive packet overhead. You should increase this to 1MB (`1048576`).

### Implementation (Client Side)

```bash
mount -t nfs -o rsize=1048576,wsize=1048576,noatime,async,hard,intr 10.0.0.1:/data /mnt
```

## Troubleshooting Key Considerations

- **The 'Sync' performance killer**: If you export with `sync`, every write must be committed to the physical disk before the server acknowledges. This is safe but **extremely slow**. If you have a battery-backed RAID controller or ZFS with an Optane SLOG, you can use `async` safely, or tune the client to use `async` mounts.
- **NFS v4.1+ and Session Trunking**: If you have multiple NICs, use NFS v4.1 or v4.2. It supports better concurrency and "Session Trunking," which is similar to multi-path I/O.
- **The 'stale file handle' ghost**: This usually happens when a directory is removed/recreated on the server while the client has it open. On a production ready system, always use the `hard,intr` mount options to ensure the client retries instead of just "hanging" the application.

## Summary

NFS is a 40-year-old protocol that still powers most of the world's Linux clusters. Its default settings are tuned for the 1990s. By increasing thread counts and aligning buffer sizes to modern network speeds, you can unlock the true performance of your storage infrastructure.
