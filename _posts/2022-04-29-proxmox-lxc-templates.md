---
layout: post
title: "Proxmox - LXC AppArmor: Resolving 'Permission Denied' on Mounts and Nesting"
date: 2022-04-29 09:53:33 +01
categories: proxmox security
tags: lxc apparmor proxmox troubleshooting containers security
---

## The Issue: The Root User who isn't Root

You're running an LXC container on Proxmox. You try to mount an NFS share, start a Docker daemon inside the container, or even just run `systemd-timesyncd`, and you get a "Permission Denied" error. You check `ls -l` and you are definitely `root`. What's happening?

## The Root Cause: AppArmor and User Namespaces

Proxmox LXC containers are (by default) **unprivileged**. This means `root` (UID 0) inside the container is mapped to a high-number UID (e.g., 100000) on the host. Furthermore, Proxmox applies a strict **AppArmor profile** to every container to prevent it from escaping to the host.

## The Ideal Fix: Precise Feature Delegation

Instead of the "nuclear option" (making the container privileged), you should delegate exactly the features the container needs.

### 1. Enabling Nesting (For Docker-in-LXC)

If you want to run Docker inside LXC, the container needs to manage its own cgroups and AppArmor profiles.

- **GUI**: Container -> Options -> Features -> Edit -> Check "Nesting".
- **CLI**: `pct set <ID> --features nesting=1`

### 2. Mounting NFS/CIFS

Standard AppArmor profiles block the `mount` syscall.

- **The proper way**: Mount the share on the **Proxmox Host** and then use a "Bind Mount" to pass it into the container.
  ```bash
  pct set <ID> -mp0 /mnt/pve/nfs-share,mp=/data
  ```

### 3. Custom AppArmor (The Last Resort)

If a specific service (like a legacy database) requires raw access that the standard profile blocks, you can switch to the `unconfined` profile (not recommended for production).

```text
# /etc/pve/lxc/<ID>.conf
lxc.apparmor.profile: unconfined
```

## Troubleshooting Key Considerations

- **The /proc and /sys Trap**: Unprivileged containers cannot modify many files in `/proc` (like `/proc/sys/net/ipv4/ip_forward`). If your app needs to change kernel parameters, you must do it on the **host** node.
- **ID Mapping**: If you use bind mounts, you'll notice that the files appear to be owned by `nobody:nogroup` inside the container. You must manually map the UIDs in the `.conf` file or use `chown` on the host to the mapped UID (e.g., `100000`).
- **dmesg is your friend**: Whenever you get a strange "Permission Denied" in LXC, run `dmesg -w` on the **Proxmox host**. You will see lines like `apparmor="DENIED" operation="mount"`. This tells you exactly what syscall is being blocked.

## Summary

LXC security is a layers-of-an-onion approach. Understanding the interaction between user namespaces and AppArmor allows you to provide the functionality your containers need without compromising the security of your entire virtualisation host.
