---
layout: post
title: "Linux - Advanced Bash: Diagnosing 'Too many open files' with lsof and Procfs"
date: 2021-04-18 17:27:40 +01
categories: linux administration
tags: bash troubleshooting lsof performance linux
---

## The Problem: The File Descriptor Leak

You have a long-running Bash script or a custom service that suddenly fails with the error: `bash: /dev/null: Too many open files`. Even after increasing the system-wide `ulimit`, the error returns after a few days. This is a classic file descriptor (FD) leak.

## Understanding the Leak in Bash

In Bash, a leak often occurs when using process substitution `<(command)` or redirects inside a loop without closing the resulting file descriptor. Each time the loop runs, a new entry is created in the process's FD table.

## Step 1: Quantifying the Leak

Use `lsof` (List Open Files) to see exactly what the process is holding.

```bash
# Count total open files for a PID
lsof -p <PID> | wc -l

# Identify the types of files
lsof -p <PID> | awk '{print $4}' | sort | uniq -c
```

If you see hundreds of entries marked as `FIFO` or `pipe`, you've found your leak.

## Step 2: The Ideal Fix – Explicit FD Management

Instead of relying on Bash to clean up, manage your descriptors explicitly using `exec`.

### The Leaky Way

```bash
while true; do
    read line < <(some_command) # Creates a subshell and a pipe every time
    # ... process line ...
done
```

### The Advanced Way

```bash
# Create a dedicated FD for the command
exec 3< <(some_command)

while read -u 3 line; do
    # ... process line ...
done

# Close the FD when finished
exec 3<&-
```

## Tips & Tricks

- **Procfs Deep Dive**: If `lsof` is not installed, you can use the kernel's own accounting: `ls -l /proc/<PID>/fd`. This shows exactly where each FD points.
- **The 'Deleted' File Trap**: `lsof` will show files marked as `(deleted)`. This happens when a process holds a handle to a file that has been `rm`'d. The disk space won't be freed until the process closes the handle. This is a common cause of "Ghost Disk Usage."
- **System-wide Auditing**: To find the top FD consumers on a host:
  ```bash
  ls /proc/*/fd | cut -d/ -f3 | sort | uniq -c | sort -rn | head
  ```

## Summary

File descriptor management is often overlooked in scripting. By moving from implicit redirects to explicit `exec` handles and monitoring via `procfs`, you can build scripts that run for months without exhausting system resources.
