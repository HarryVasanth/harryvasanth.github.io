---
layout: post
title: "Linux - Performance: Deep Observability with bpftrace and Custom eBPF Scripts"
date: 2024-01-15 20:35:03 +01
categories: linux performance
tags: linux ebpf bpftrace performance troubleshooting observability administration
---

## The Limits of Traditional Observability

In the modern infrastructure stack, standard tools like `top`, `iostat`, and `tcpdump` often fail to answer the most critical question: _Why_ is a specific request slow? These tools provide aggregates and snapshots, but they cannot show you the internal latency of a specific kernel function or the exact path a packet takes through the XDP (Express Data Path).

**eBPF (Extended Berkeley Packet Filter)** has fundamentally changed Linux observability by allowing us to run sandboxed code directly in the kernel. While tools like Netdata provide excellent high-level heatmaps, a experienced administrator needs the ability to write ad-hoc, surgical queries to diagnose "impossible" bugs. This is where **bpftrace** becomes indispensable.

## bpftrace: The DTrace for Linux

`bpftrace` uses a high-level domain-specific language (DSL) inspired by AWK and C. It compiles your scripts into eBPF bytecode and executes them within the kernel, providing a safe and efficient way to instrument almost any kernel or user-space event.

### 1. The Anatomy of a bpftrace Probe

A bpftrace script consists of three parts:

- **The Probe**: Where to hook (e.g., `kprobe:vfs_read`, `tracepoint:syscalls:sys_enter_openat`).
- **The Filter**: Optional logic to limit which events are captured (e.g., `/pid == 1234/`).
- **The Action**: What to do when the probe fires (e.g., `{ @[comm] = count(); }`).

## Technical Scenario 1: Identifying "Leaky" File Descriptors

Imagine a production service is slowly hitting its `nofile` limit. You know it's opening files, but you don't know _which_ files or if it's closing them correctly.

```bpftrace
# Track file opens and prints the filename and the return descriptor
tracepoint:syscalls:sys_enter_openat
/comm == "my-app-name"/
{
    printf("Open: %s\n", str(args->filename));
}

tracepoint:syscalls:sys_enter_close
/comm == "my-app-name"/
{
    printf("Close FD: %d\n", args->fd);
}
```

## Technical Scenario 2: Measuring Block I/O Latency Histograms

Aggregate disk latency metrics often hide "tail" latency. We can use bpftrace to create a power-of-two histogram of every single block I/O request.

```bpftrace
kprobe:vfs_read
{
    @start[tid] = nsecs;
}

kretprobe:vfs_read
/@start[tid]/
{
    $lat = nsecs - @start[tid];
    @latency_ns = hist($lat);
    delete(@start[tid]);
}
```

This script gives you an immediate, visual distribution of your storage performance. If you see a "bimodal" distribution (two peaks), it's a clear indicator that your storage backend is struggling with specific block sizes or cache misses.

## Phase 3: Uprobes (User-Space Probes)

One of the most powerful features of bpftrace is the ability to instrument **user-space** applications without restarting them. You can hook into specific functions in a library (like `ssl_read` in OpenSSL) to see raw traffic before it is encrypted, provided you have the appropriate symbols and permissions.

```bpftrace
# Count SSL read calls by process name
uprobe:/usr/lib/x86_64-linux-gnu/libssl.so.3:SSL_read
{
    @[comm] = count();
}
```

## Suitable Strategy: Production Safety and 'One-Liners'

While bpftrace is efficient, it is not "free." Probes on high-frequency events (like `sched_switch` on a 64-core box) can add measurable overhead.

- **The Pro-Tip**: Always use filters to narrow your scope as much as possible.
- **The 'One-Liner'**: For quick checks, use the CLI directly:

```bash
# Show which processes are opening which files, system-wide
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s is opening %s\n", comm, str(args->filename)); }'
```

## Summary: Moving from Guesswork to Proof

The hallmark of a experienced system administrator is the refusal to rely on "restarts" as a permanent fix. By mastering bpftrace, you gain the ability to prove exactly where a bottleneck lies, whether it's a kernel lock, a disk buffer, or a misbehaving application thread. It is the ultimate "truth-seeking" tool in the Linux ecosystem, allowing you to build and maintain infrastructure with surgical precision.
