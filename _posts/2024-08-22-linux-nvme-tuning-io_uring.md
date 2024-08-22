---
layout: post
title: "Linux - Storage: Optimising NVMe Performance with io_uring and IRQ Affinity"
date: 2024-08-22 08:34:33 +01
categories: linux performance
tags: linux performance nvme io_uring storage tuning administration
---

## The NVMe Bottleneck: The Kernel Itself

Modern NVMe drives are capable of millions of IOPS (I/O Operations Per Second) and sub-100 microsecond latency. However, many Linux servers fail to achieve even 50% of this potential. The reason is that the traditional Linux I/O stack (using `read/write` or `aio`) was designed for the era of spinning disks and slow SSDs. At the speeds of modern NVMe, the overhead of "Context Switching" between user-space and kernel-space becomes the primary bottleneck.

A experienced administrator knows that to unlock the true power of NVMe, we must bypass these legacy bottlenecks using **io_uring** and **IRQ Affinity**.

## 1. The io_uring Revolution

Introduced in kernel 5.1, `io_uring` is a high-performance asynchronous I/O interface. Unlike older methods, it uses a pair of "Ring Buffers" shared between the application and the kernel. This allows the application to submit thousands of I/O requests and retrieve the results without a single system call.

**How to leverage it**:
Most modern performance-critical applications (Nginx, MariaDB 10.6+, PostgreSQL 15+, and Vector) now support `io_uring` natively.

- **In MariaDB**: Set `innodb_use_native_aio=OFF` and ensure your OS has the `liburing` library.
- **In Nginx**: Use the `aio io_uring;` directive in your configuration.

## 2. Managing IRQ Affinity (Interrupt Handling)

When an NVMe drive completes a write, it sends an "Interrupt" (IRQ) to the CPU. By default, the Linux `irqbalance` daemon tries to spread these interrupts across all CPU cores. While this sounds good, it's actually bad for performance because it causes "Cache Misses" as the data is moved from core to core.

**The Ideal Fix**: Bind the NVMe interrupts to the specific CPU cores physically closest to the PCIe lane of the drive (the "Local NUMA Node").

### Identifying the NUMA Node

```bash
# Find which NUMA node the NVMe is attached to
cat /sys/block/nvme0n1/device/numa_node
```

### Manual Binding (Advanced)

If `irqbalance` is not performing well, you can manually set the affinity mask in `/proc/irq/`.

```bash
# Example: Bind IRQ 125 to CPU core 0
echo 1 > /proc/irq/125/smp_affinity_list
```

## 3. NVMe-Specific sysctl Tuning

The default Linux block layer settings are often too conservative for NVMe.

```text
# Increase the maximum number of requests in the I/O queue
echo 1024 > /sys/block/nvme0n1/queue/nr_requests

# Set the I/O scheduler to 'none' (NVMe drives handle their own scheduling)
echo none > /sys/block/nvme0n1/queue/scheduler

# Increase the number of allowable open files (descriptors)
fs.file-max = 2097152
```

## 4. Polling vs. Interrupts (The Extreme Option)

For ultra-low latency, you can enable "Hybrid Polling." Instead of the CPU waiting for an interrupt, it actively "polls" the NVMe drive to see if the task is done. This consumes more CPU but reduces latency by several microseconds.

```bash
# Enable I/O polling on the NVMe driver
echo 1 > /sys/module/nvme/parameters/poll_queues
```

## Summary: From Hardware to Throughput

Hardware is only as fast as the software that manages it. By moving to `io_uring`, aligning your IRQs with your NUMA topology, and removing the unnecessary I/O scheduler, you can transform your Linux server from a bottleneck into a high-speed data engine. This level of tuning is what allows a single server to handle the workloads that previously required an entire rack of equipment. It is the hallmark of a high-performance infrastructure engineer in 2024.
