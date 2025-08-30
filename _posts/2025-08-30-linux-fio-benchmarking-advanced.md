---
layout: post
title: "Linux - Benchmarking: Using 'FIO' for Real-World Storage Performance Profiles"
date: 2025-08-30 09:22:54 +01
categories: linux performance
tags: linux performance storage fio benchmarking administration
---

## The Fallacy of 'Sequential' Speed Tests

When buying a new SSD, you often see "5000 MB/s" advertised. These are sequential speeds-reading one giant file from start to finish. In a real-world server environment (a database or a hypervisor), this number is almost meaningless. A database performs thousands of tiny, random 4KB writes. A web server performs many tiny, random 16KB reads.

To understand how your storage will actually behave in production, you need **FIO (Flexible I/O Tester)**. FIO is the industry standard for creating complex, reproducible storage benchmarks.

## Phase 1: The 'Random 4K' Test (The Database Profile)

This test simulates the intense random pressure of a transactional database like PostgreSQL or MariaDB.

```bash
fio --name=random-write --ioengine=liburing --rw=randwrite --bs=4k \
    --size=4G --numjobs=1 --runtime=60 --time_based --group_reporting \
    --iodepth=64 --direct=1 --filename=/dev/nvme0n1
```

- `--ioengine=liburing`: Uses the high-performance Linux kernel interface we discussed in 2024.
- `--iodepth=64`: Simulates 64 concurrent I/O requests.
- `--direct=1`: Bypasses the OS cache to test the raw hardware speed.

## Phase 2: Analyzing the 'Tail' (Latency Percentiles)

FIO doesn't just give you an "Average." It gives you a distribution. Look for the **clat** (Completion Latency) section in the output:

```text
clat percentiles (msec):
 |  1.00th=[ 0.01],  5.00th=[ 0.02], 50.00th=[ 0.05], 90.00th=[ 0.12],
 | 95.00th=[ 0.18], 99.00th=[ 0.45], 99.90th=[ 1.20], 99.99th=[ 5.50]
```

**The Advanced Insight**: The `99.99th` percentile is the most important. It tells you that 0.01% of your requests are taking 5.5ms while the average is 0.05ms. These are your "Tail Latencies" that cause application timeouts.

## Phase 3: The 'Mixed' Workload (The Real-World Profile)

Most servers perform both reads and writes simultaneously. A 75% read / 25% write split is a common "General Server" profile.

```bash
fio --name=mixed-load --rw=randrw --rwmixread=75 --bs=16k \
    --ioengine=liburing --iodepth=32 --direct=1 --size=10G
```

## Phase 4: Production Benchmarking (Safety First)

Never run FIO on a live database partition with `--direct=1` and `--rw=randwrite`. You will corrupt your data.

- **The Safe Way**: Run FIO against a temporary file within the filesystem.

```bash
--filename=/data/testfile.fio
```

## Summary: From Guesswork to Data

FIO is more than a benchmarking tool; it is a "Storage Microscope." By creating custom profiles that match your application's behavior, you can prove why a specific disk is failing, justify the purchase of high-end NVMe drives, or verify that your ZFS/RAID settings are optimal. In a professional environment, "the disk feels slow" is a complaint; an FIO report is a diagnostic proof.
