---
layout: post
title: "Docker - Security: Implementing 'Rootless' Mode for Maximum Host Protection"
date: 2025-05-18 10:28:13 +01
categories: docker security
tags: docker rootless security containers hardening administration
---

## The Inherent Risk of the Docker Daemon

For years, the Docker daemon (`dockerd`) was a security concern because it required root privileges to run. If an attacker escaped a container, they had a direct path to the root account of the host machine. While Docker introduced many security features (Seccomp, AppArmor), the root-level daemon remained a significant "Single Point of Failure."

**Rootless Docker** (fully mature in 2024-2025) allows you to run the Docker daemon and containers as an unprivileged user. This uses "User Namespaces" to map the root user _inside_ the container to a non-privileged user _outside_ the container. If an attacker escapes, they find themselves trapped as a "nobody" user on the host.

## The Architectural Challenge: The 'Privileged' Port

One of the biggest obstacles to rootless mode is networking. On Linux, a non-root user is not allowed to bind to "privileged ports" (any port below 1024). If your web server needs port 80 or 443, a standard rootless setup will fail.

**The Optimal Solution**: We must modify the kernel's `unprivileged_port_start` parameter.

```bash
# Allow non-root users to bind to ports from 80 upwards
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80
```

To make this persistent:

```bash
echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee -a /etc/sysctl.d/99-rootless-docker.conf
```

## Implementation: Setting up Rootless Mode

### 1. Prerequisites: Slirp4netns

Since a rootless user doesn't have permissions to create real network bridges, Docker uses `slirp4netns` for user-mode networking.

```bash
sudo apt-get install -y dbus-user-session slirp4netns uidmap
```

### 2. Run the Install Script

The rootless setup is handled by a dedicated script that installs a user-level systemd service.

```bash
curl -fsSL https://get.docker.com/rootless | sh
```

### 3. Environment Configuration

You must tell the Docker CLI to use the user-level socket instead of the system-level `/var/run/docker.sock`.

```bash
# Add this to your ~/.bashrc
export PATH=/home/user/bin:$PATH
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

## Advanced Hardening: Managing Sub-UIDs and Sub-GIDs

Rootless mode depends on the host having a range of "Subordinate UIDs" available for the user.

```bash
# /etc/subuid
user:100000:65536
```

This tells the kernel that the user `user` is allowed to manage 65,536 UIDs starting from 100,000. Each container you start will consume a slice of this range, ensuring that even different containers on the same host are isolated from each other.

## Why This is the Standard for 2025

- **Host Safety**: Even a total container breakout cannot compromise the host's root filesystem.
- **Compliance**: Many modern security standards (like SOC2) now mandate the use of non-root daemons for container orchestration.
- **Multi-User Servers**: Multiple developers can run their own independent Docker daemons on the same shared server without interfering with each other.

## Summary

Security is about defense in depth. By moving your production Docker workloads to rootless mode, you remove the most significant risk in containerised infrastructure. While it requires minor kernel tuning for port access, the peace of mind knowing that an exploit won't lead to a total host takeover is worth the effort. This is the definitive standard for any experienced administrator managing containers in a security-conscious environment.
