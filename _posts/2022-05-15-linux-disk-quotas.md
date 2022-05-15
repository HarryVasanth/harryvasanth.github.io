---
layout: post
title: "Linux - Disk Quotas: Solving 'Mountpoint not found' and systemd Race Conditions"
date: 2022-05-15 10:34:55 +01
categories: linux storage
tags: quotas troubleshooting linux storage administration
---

## The Problem: The Zombie Quota

You've added `usrquota,grpquota` to your `/etc/fstab`. You've remounted the filesystem. But when you run `quotacheck -avug`, you get: `quotacheck: Cannot find filesystem to check or filesystem not mounted with quota option`. You've triple-checked `mount | grep quota` and it's definitely there.

## The Root Cause: systemd and the Quota Service

On modern Linux distributions (Ubuntu 20.04+, Debian 11+, RHEL 8+), the initialization of quotas is handled by `systemd`. There is often a race condition where the kernel has enabled the quota flags on the mount, but the quota user-space tools can't "hook" into the mount because the `aquota.user` files don't exist yet, or are being blocked by a stale lock.

## The Ideal Fix: The Manual Re-initialisation

When the automated tools fail, you must perform the ritual of manual initialisation.

### 1. Force the remount

Sometimes a simple `mount -o remount` isn't enough to trigger the quota subsystem in the kernel.

```bash
sudo mount -o remount,usrquota,grpquota /home
```

### 2. Create the Index Files Manually

The `quotacheck` tool needs the binary index files to exist in the root of the partition.

```bash
sudo touch /home/aquota.user /home/aquota.group
sudo chmod 600 /home/aquota.*
```

### 3. The 'Master' Check Command

Use the `-m` (don't remount read-only) and `-u/g` flags specifically:

```bash
sudo quotacheck -cum /home
sudo quotaon -v /home
```

## Tips & Tricks

- **XFS is Different**: If you are using XFS (default on RHEL/CentOS), **do not use quotacheck**. XFS manages quotas internally. You must enable them at mount time via the `rootflags=uquota` kernel parameter (for `/`) or in `fstab`, and use the `xfs_quota` tool for management.
- **Grace Periods**: A well-versed admin always sets a "Grace Period" (usually 7 days). This allows a user to go over their "Soft Limit" temporarily (e.g., to finish a large download) but hard-stops them at the "Hard Limit."
  ```bash
  sudo setquota -t 604800 604800 /home
  ```
- **Monitoring**: Integrate quota usage into your monitoring (Prometheus/Zabbix). Running out of quota feels like a full disk to the user, but `df -h` will show plenty of space, leading to hours of wasted troubleshooting.

## Summary

Disk quotas are a legacy technology that remains essential in multi-user environments. Their interaction with modern init systems can be flaky, but by understanding how the kernel tracks quota binary files, you can consistently resolve the "Mountpoint not found" error.
