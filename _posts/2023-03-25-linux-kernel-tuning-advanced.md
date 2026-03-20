---
layout: post
title: "Linux - Performance: Advanced Kernel Tuning with sysctl for High-Concurrency Clusters"
date: 2023-03-25 22:59:14 +01
categories: linux performance
tags: linux kernel tuning performance sysctl networking administration
---

## The Default Configuration Bottleneck

Out of the box, most Linux distributions are tuned for general-purpose workloads-a balance between a desktop and a light server. However, when you deploy a high-concurrency web server or a load balancer (like Nginx, HAProxy, or a heavy Go application), these default limits become your primary enemy. You will encounter "Connection Refused" or "Out of Socket Memory" errors long before your CPU or RAM are actually exhausted.

A experienced administrator knows that the Linux kernel's networking stack must be explicitly widened to handle tens of thousands of simultaneous connections. This guide explores the critical `sysctl` parameters that define a high-performance Linux node in a production environment.

## 1. Expanding the Connection Backlog

When a new connection arrives, it is placed in a queue before the application can `accept()` it. If this queue is full, the kernel drops the connection.

- **`net.core.somaxconn`**: The maximum number of "completely established" sockets waiting to be accepted. The default is often 128. For a high-load server, set this to **4096** or higher.
- **`net.ipv4.tcp_max_syn_backlog`**: The number of "half-open" connections (SYN received). Increasing this helps defend against SYN flood attacks and handles bursts of new users.

## 2. Speeding up Socket Reuse

A common issue on busy servers is the **TIME_WAIT** state. When a connection closes, the socket remains in a "holding pattern" for 60 seconds to ensure no late packets arrive. With high traffic, you can quickly run out of available ports.

- **`net.ipv4.tcp_tw_reuse`**: Allows the kernel to reuse a socket in the `TIME_WAIT` state for a new connection if it is safe from a protocol standpoint.
- **`net.ipv4.ip_local_port_range`**: Increase the range of available ephemeral ports.

```text
net.ipv4.ip_local_port_range = 1024 65535
```

## 3. Tuning the TCP Buffer (Window Scaling)

For high-latency or high-bandwidth links, the size of the TCP buffers determines how much data can be "in flight." If these buffers are too small, your 10Gbps link will perform like a 100Mbps link.

```text
# Increase max read/write buffer sizes (16MB)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
# TCP Autotuning: min, default, max
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
```

## 4. The 'File Descriptor' Limit

In Linux, "Everything is a file," including network sockets. The default limit for open files is often 1024, which is far too low for a web server handling modern API traffic.

While `sysctl` handles the system-wide limit (`fs.file-max`), you must also increase the per-process limits in `/etc/security/limits.conf` or your systemd service file.

```text
# /etc/security/limits.conf
* soft nofile 1048576
* hard nofile 1048576
```

## 5. Virtual Memory (Swap) Tuning

In a high-performance cluster, swapping to disk is the death of latency.

- **`vm.swappiness`**: This controls how aggressively the kernel swaps memory. For a server with plenty of RAM, reduce this from 60 to **10** or even **1**.
- **`vm.vfs_cache_pressure`**: Controls the tendency of the kernel to reclaim the memory used for caching of directory and inode objects. Setting this to 50 (from 100) helps keep filesystem metadata in RAM.

## Implementation: Applying Changes Permanently

Never apply sysctl changes directly to `/etc/sysctl.conf`. Instead, use the `/etc/sysctl.d/` directory to keep your customisations organised.

```bash
# /etc/sysctl.d/99-high-concurrency.conf
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 10000 65000
net.ipv4.tcp_slow_start_after_idle = 0
vm.swappiness = 10
vm.vfs_cache_pressure = 50
```

{: file='/etc/sysctl.d/99-high-concurrency.conf'}

Apply the changes without a reboot:

```bash
sudo sysctl --system
```

## Summary: Monitoring the Impact

Tuning is not a one-time event; it is a feedback loop. Use `netstat -s` or `ss -s` to look for "SYNs to LISTEN sockets dropped" or "TCP sockets finished time_wait." If these numbers are growing, your limits are still too low. By widening the kernel's throughput capacity, you ensure that your hardware is used to its full potential, providing a smooth and responsive experience for thousands of concurrent users. This level of kernel rigour is what separates a standard installation from a high-availability infrastructure.
