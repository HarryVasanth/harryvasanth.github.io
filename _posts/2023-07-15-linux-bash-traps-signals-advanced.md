---
layout: post
title: "Linux - Automation: Advanced Bash Traps and Signal Handling for Robust Atomic Scripts"
date: 2023-07-15 16:57:28 +01
categories: linux automation
tags: bash linux automation scripting troubleshooting administration
---

## The 'Zombie' Resource Problem

We have all written a Bash script that creates a temporary directory, starts a background process, or mounts a filesystem, only to have the script crash halfway through. The result? A `/tmp` folder that never gets deleted, or a background process that keeps running forever, consuming CPU and locking files.

In professional automation, a script must be "Atomic"-it either finishes completely, or it cleans up after itself perfectly regardless of how it failed (Ctrl+C, a crash, or a system shutdown). The secret to this is the **`trap`** command.

## How 'trap' Works: The Signal Pipeline

The `trap` command tells the shell to execute a specific function or command when it receives a signal.

- **EXIT**: The script is finishing (either naturally or via `exit`).
- **SIGINT**: The user pressed Ctrl+C.
- **SIGTERM**: A system manager (like systemd) is asking the script to stop.
- **ERR**: A command within the script failed (requires `set -e`).

## Implementation: The 'Perfect' Temporary Workspace

```bash
#!/bin/bash
# Advanced Bash Boilerplate: exit on error, fail on unset variables, fail on pipe error
set -euo pipefail

# Define the cleanup function
cleanup() {
    # Capture the exit code of the last command
    local exit_code=$?
    echo "Cleaning up... (Exit code: $exit_code)"

    # Remove temporary files
    if [[ -d ${TEMP_DIR:-} ]]; then
        rm -rf "$TEMP_DIR"
    fi

    # If we started a background process, kill it
    if [[ -n ${BG_PID:-} ]]; then
        kill "$BG_PID" 2>/dev/null || true
    fi

    # Final exit with the original code
    exit "$exit_code"
}

# Register the trap: Run cleanup on EXIT, INT, or TERM
trap cleanup EXIT INT TERM

# Create a temporary workspace
TEMP_DIR=$(mktemp -d)
echo "Working in $TEMP_DIR"

# Start a 'dummy' background task
sleep 100 &
BG_PID=$!

# Simulate a failure
if [[ ${1:-} == "fail" ]]; then
    echo "Simulating a crash!"
    exit 1
fi

echo "Task complete!"
```

## Advanced Logic: The 'Double Trap' and Lockfiles

Sometimes, your cleanup process itself might take time (e.g., unmounting a busy network drive). What if the user presses Ctrl+C a _second_ time during the cleanup?
**The Ideal Fix**: Redefine the trap inside the cleanup function to ignore further signals.

```bash
cleanup() {
    # Ignore further signals during cleanup to prevent interruption
    trap '' INT TERM
    echo "Performing slow cleanup..."
    umount /mnt/backup_storage
    exit
}
```

### Implementing Mutual Exclusion (Locking)

To prevent a script from running twice at the same time:

```bash
LOCKFILE="/tmp/myscript.lock"
if ! exec 3>"$LOCKFILE" 2>/dev/null; then
    echo "Error: Script is already running."
    exit 1
fi
# Ensure the lock is released on exit
trap 'rm -f "$LOCKFILE"' EXIT
```

## Real-World Use Case: Database Backups

When performing a database backup, you often need to lock tables or create a filesystem snapshot. If your script crashes while the tables are locked, your application will go offline.

```bash
lock_database
trap 'unlock_database' EXIT INT TERM

# Perform the backup
tar -czf backup.tar.gz /var/lib/mysql/data
```

With this trap, even if the disk fills up or the SSH connection drops, the `EXIT` trap will ensure the `unlock_database` command runs, saving your production environment from a permanent hang.

## Summary

Reliability in Linux administration isn't about writing code that never fails; it's about writing code that fails gracefully. By mastering Bash traps and signal handling, you ensure that your automation scripts never leave your systems in an inconsistent state. This level of defensive programming is what separates a "quick script" from a production-grade infrastructure tool. It is a mandatory skill for anyone managing critical servers.
