---
layout: post
title: "Linux - Recovery: Implementing Timeshift for Bulletproof System Restores"
date: 2024-05-18 03:25:49 +01
categories: linux administration
tags: timeshift linux recovery backups administration rsync btrfs
---

## The 'Update Anxiety' in Linux Administration

Every administrator has experienced the "broken update." You run `apt upgrade`, the kernel or a critical library updates, and suddenly the system fails to boot or a key service crashes. In a professional environment, "fixing it manually" while the service is down is not a strategy. You need a way to revert the _entire operating system_ to a known-good state in seconds.

**Timeshift** provides this capability. Unlike standard backup tools (like `rsync` or `tar`) which focus on your data, Timeshift focuses on the **System State**. It functions like "System Restore" on Windows or "Time Machine" on macOS, but with the power and flexibility of the Linux filesystem.

## Two Modes of Operation: RSYNC vs BTRFS

### 1. RSYNC Mode (The Universal Choice)

Timeshift uses `rsync` and hard-links to create incremental snapshots.

- **Pros**: Works on any filesystem (EXT4, XFS).
- **Cons**: Takes time to copy data, and takes up disk space for every changed file.

### 2. BTRFS Mode (The Ideal Choice)

If your OS is installed on a BTRFS partition, Timeshift uses the filesystem's native snapshotting capability.

- **Pros**: Snapshots are instantaneous and take zero extra space until files are modified. You can snapshot a 1TB OS in 0.1 seconds.
- **Cons**: Requires a specific BTRFS subvolume layout (`@` and `@home`).

## Implementation: Setting up Automated Snapshots

### 1. Installation

```bash
sudo apt update && sudo apt install timeshift
```

### 2. Configuration via CLI

For a headless server, we use the command-line interface to set our schedule.

```bash
# Set snapshot device and schedule (5 daily, 3 weekly)
sudo timeshift --create --comments "Before major update" --tags D
```

## The Suitable Workflow: 'Pre-Flight' Snapshots

A professional administrator never performs a major change without a snapshot. Before installing a new database version or changing a complex networking config:

```bash
# Create an on-demand snapshot
sudo timeshift --create --comments "Pre-DB-Upgrade"

# Perform your risky task
sudo apt upgrade mysql-server
```

**If things go wrong**:

```bash
# List available snapshots
sudo timeshift --list

# Restore the specific snapshot
sudo timeshift --restore --snapshot '2024-05-18_14-00-01'
```

Timeshift will prompt you to confirm, and then it will overwrite the system files with the versions from the snapshot. A quick reboot later, and you are back to exactly where you were before the failure.

## Troubleshooting: Restoring from a Live USB

If the system is so broken it won't even boot to a shell:

1.  Boot from a standard Ubuntu/Debian Live USB.
2.  Install Timeshift in the live environment (`sudo apt install timeshift`).
3.  Launch Timeshift, and it will automatically find your snapshots on the internal disk.
4.  Run the restore. Timeshift will handle the mounting and re-installation of the GRUB bootloader automatically.

## Summary: Peace of Mind for the Administrator

System administration is stressful enough without the fear of permanent failure. Timeshift provides a "Safety Net" that allows you to experiment, update, and refactor your systems with confidence. By implementing an automated snapshot schedule and a "pre-flight" manual snapshot policy, you ensure that your "Mean Time To Recovery" (MTTR) is measured in minutes, not hours. It is an essential tool for any Linux administrator who values their uptime (and their sleep).
