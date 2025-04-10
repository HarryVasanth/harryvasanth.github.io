---
layout: post
title: "Linux - Debugging: Advanced Kernel Panic Post-Mortems with kdump, crash and Debug Symbols"
date: 2025-04-10 04:40:45 +01
categories: linux troubleshooting
tags: linux kernel troubleshooting kdump debugging administration
---

## The 'Black Hole' of a Kernel Panic

A kernel panic is the most severe failure a Linux system can experience. The OS has encountered an internal error so critical that it cannot safely continue. The screen shows a "Blinkenlights" crash log, and the system becomes unresponsive. For a junior administrator, the only solution is a hard reboot. For a experienced administrator, a reboot is only the beginning. You need to know _why_ it happened to prevent it from happening again.

To do this, you must capture a **vmcore** (a full dump of the system's RAM at the moment of the crash) and analyze it using the **crash** utility.

## Phase 1: The 'Second' Kernel (kdump)

How can a crashed kernel write its memory to a disk? It can't. It's crashed.
**The Solution**: **kdump**. When the system boots, it reserves a small slice of RAM (e.g., 256MB) that is hidden from the main OS. Inside this slice, a tiny, "crash kernel" is pre-loaded. When the main kernel panics, it immediately hands over control to this crash kernel, which has its own drivers and can safely write the contents of the _main_ RAM to the disk.

### 1. Installation and Reservation

```bash
sudo apt install kdump-tools crash kexec-tools
```

In `/etc/default/grub`, ensure you have the reservation:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet crashkernel=256M"
```

After editing, run `sudo update-grub` and reboot.

### 2. Configuration

Edit `/etc/default/kdump-tools`. Ensure `KDUMP_SYSCTL="kernel.panic_on_oops=1"` is set to ensure the system dumps memory even on minor "Oops" events.

## Phase 2: Analyzing the vmcore

Once a crash has occurred, you will find a large file (the size of your RAM) in `/var/crash/`.

### 1. Launching the Crash Utility

You need the kernel image and the debug symbols (dbgsym) for your exact kernel version.

```bash
sudo crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/2025-04-20-10:00/dump.202504201000
```

### 2. The 'Advanced' Commands for Root Cause Analysis

- **`bt` (Backtrace)**: Shows the call stack of the process that was running when the kernel died. This is the "Smoking Gun."
- **`log`**: Displays the internal kernel log buffer (dmesg) leading up to the crash.
- **`ps`**: Shows the state of all processes at the time of the crash. Look for processes in the `D` (Uninterruptible Sleep) state.
- **`kmem -i`**: Shows the memory usage. Was the crash caused by a "NULL Pointer Dereference" or a "Stack Overflow"?

## Real-World Scenario: The 'Buggy Driver'

You see `bt` ends in `bnxt_en` (a Broadcom network driver).

```text
#0 [ffffb06180373b58] bnxt_rx_pkt at ffffffffc0456210 [bnxt_en]
#1 [ffffb06180373bc0] bnxt_poll_nitro_rx at ffffffffc0457a44 [bnxt_en]
```

You have now proven that the crash was caused by the network driver, likely during a high-traffic event. You can now search for specific firmware updates or driver bug reports rather than guessing and "reinstalling the OS."

## Summary: Post-Mortem Rigour

A server crash is a data point. By implementing kdump, you ensure that you never lose that data. Analysis of kernel panics is one of the most difficult skills in Linux administration, but it is also the most rewarding. It moves you from "hope-based administration" to "evidence-based engineering." In a high-availability environment, a well-versed admin who can read a backtrace is worth their weight in gold. It represents the move from "Reactive Rebooting" to "Surgical Debugging and Prevention."
