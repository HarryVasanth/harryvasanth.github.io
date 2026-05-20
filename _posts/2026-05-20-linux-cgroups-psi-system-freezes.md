---
layout: post
title: "Linux - Administration: Mastering Cgroups v2 and PSI to Eradicate System Freezes"
date: 2026-05-20 14:15:00 +01
categories: linux administration
tags: linux kernel cgroups troubleshooting systemd
---

## The Legacy 'Out of Memory' Lockup

Every Linux administrator has experienced the dreaded memory starvation lockup. A rogue database query or a memory-leaking application consumes all available RAM. The system begins to swap aggressively. Disk I/O skyrockets, and the CPU spends all its cycles simply moving pages in and out of memory (thrashing).

During this phase, the server becomes completely unresponsive. You cannot establish an SSH connection. The kernel's built-in Out of Memory (OOM) killer is supposed to intervene and terminate the offending process, but because the kernel itself is bogged down by the thrashing, the OOM killer might take hours to trigger. By the time it acts, your production service has experienced a catastrophic outage.

In 2026, relying on the legacy kernel OOM killer is considered an operational failure. A professional infrastructure environment now relies on **Cgroups v2**, **Pressure Stall Information (PSI)**, and **systemd-oomd** to preemptively resolve memory starvation before the system freezes.

## The Kernel Foundation: Pressure Stall Information (PSI)

For decades, we measured system load using simple averages (the numbers you see in `top` or `uptime`). These aggregates do not tell you _why_ a system is slow.

The Linux kernel now exposes **Pressure Stall Information (PSI)**. PSI measures the exact percentage of wall-clock time that tasks spend waiting for hardware resources. It splits this data into three categories: CPU, Memory, and I/O.

You can view this data directly from the kernel virtual filesystem:

```bash
cat /proc/pressure/memory

```

The output provides metrics for 10-second, 60-second, and 300-second windows:
`some avg10=15.20 avg60=5.10 avg300=1.00 total=456789`
`full avg10=2.10 avg60=0.50 avg300=0.10 total=12345`

- **Some**: The percentage of time where _at least one_ task was delayed due to a lack of memory.
- **Full**: The percentage of time where _all_ non-idle tasks were delayed simultaneously. If "full" starts rising above 0, your system is actively thrashing.

## Phase 1: Enabling PSI and Cgroups v2

Most modern distributions default to Cgroups v2, but PSI is sometimes disabled to save a negligible amount of CPU overhead. As an administrator, you must explicitly enable it in your bootloader.

Edit your GRUB configuration:

```bash
sudo nano /etc/default/grub

```

Append `psi=1` to the command line parameters:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet psi=1 systemd.unified_cgroup_hierarchy=1"

```

Update GRUB and reboot the server to apply the new kernel parameters.

## Phase 2: The User-Space OOM Killer (systemd-oomd)

With PSI providing high-resolution metrics, we can implement a "User-Space OOM Killer". The standard tool for this is `systemd-oomd`.

Instead of waiting for the system to run entirely out of memory, `systemd-oomd` constantly monitors the PSI metrics. If it sees that memory pressure is exceeding a safe threshold (for example, tasks are stalled 60% of the time over a 10-second window), it takes immediate action and kills the control group (cgroup) consuming the most memory.

### 1. Installation and Global Configuration

Ensure the daemon is installed and enabled:

```bash
sudo apt update && sudo apt install systemd-oomd
sudo systemctl enable --now systemd-oomd

```

### 2. Targeting Specific Slices

The true power of `systemd-oomd` is that it respects the Cgroups v2 hierarchy. You do not want it to kill critical system components (like `sshd` or `networkd`); you want it to target user sessions or specific application services.

We configure this by modifying the properties of a specific systemd "slice". For example, to allow `systemd-oomd` to manage background services running in the `system.slice`:

```bash
sudo systemctl edit system.slice

```

Add the following directives:

```ini
[Slice]
# Enable OOMD management for this slice
ManagedOOMMemoryPressure=kill
# Set the pressure threshold (e.g. 60% memory pressure over 20 seconds)
ManagedOOMMemoryPressureLimit=60%

```

Apply the changes:

```bash
sudo systemctl daemon-reload

```

## Advanced Strategy: Protecting Critical Daemons

If `systemd-oomd` decides a slice is under too much pressure, it will terminate the largest cgroup within that slice. However, there are often "helper" services that consume memory but must never be killed.

You can protect specific services by assigning them an OOM preference. For instance, if you are running a mission-critical PostgreSQL database alongside a less important Redis cache, you instruct the system to heavily penalise the cache before touching the database.

Edit the Redis service override:

```bash
sudo systemctl edit redis-server.service

```

```ini
[Service]
# Tell systemd-oomd to target this service first during high pressure
OOMPolicy=continue

```

Conversely, for PostgreSQL:

```ini
[Service]
# Tell systemd-oomd to avoid killing this service at all costs
OOMScoreAdjust=-900

```

## Summary

The combination of Cgroups v2, PSI, and `systemd-oomd` represents a fundamental shift in Linux systems administration. By moving OOM decisions from a panicked, thrashing kernel into a calculated, metric-driven user-space daemon, you ensure that memory exhaustion events result in the clean termination of a single service rather than the total unresponsiveness of the entire host. This preemptive approach to resource starvation is a mandatory configuration for any administrator focused on maintaining high-availability enterprise infrastructure in 2026.
