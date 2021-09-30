---
layout: post
title: "Linux - Monit: Implementing File Integrity Monitoring (FIM) for Critical Configs"
date: 2021-09-30 18:07:38 +01
categories: linux security
tags: monit security hardening monitoring linux
---

## The Silent Compromise

Attackers rarely change a system's behaviour immediately. Instead, they modify a legitimate configuration file (like `/etc/ssh/sshd_config` or `/etc/pam.d/common-auth`) to create a backdoor for later use. Standard monitoring only checks if the service is "Up," which it is. You need to monitor the **integrity** of the files themselves.

## The Optimal Solution: Monit Checksums

Monit has a lightweight, built-in File Integrity Monitoring (FIM) engine that can track changes to file attributes and content (via MD5/SHA1 hashes).

### Implementation

Add this to your Monit configuration directory:

```conf
check file sshd_bin with path /usr/sbin/sshd
    if failed permission 755 then alert
    if failed uid root then alert
    if failed checksum then alert

check file sshd_config with path /etc/ssh/sshd_config
    if failed checksum then alert
    if failed permission 644 then alert
    # Automatically restart service if config changes (useful for automation)
    exec "/usr/bin/systemctl reload ssh"
```

{: file='/etc/monit/conf.d/ssh-integrity'}

## How it Works

The first time Monit runs, it calculates the hash of the file and stores it in its local state database (`/var/lib/monit/state`). On every subsequent cycle, it re-calculates the hash. If they don't match, it triggers an alert.

## Tips & Tricks

- **The 'False Positive' Trap**: If you use configuration management (like Ansible or Puppet), you **will** trigger these alerts every time you push a legitimate change.
  - **The Fix**: Include a task in your Ansible playbook to temporarily disable Monit alerts before the config push, or to run `monit reload` immediately after to update the state.
- **Monitoring Directories**: Monit can also monitor entire directories for new or deleted files, which is excellent for monitoring `/var/www/html` for uploaded webshells.
- **Performance**: Calculating SHA256 hashes for large files every minute can impact I/O. Use FIM only for small, critical configuration files and binaries. For large-scale data integrity, use ZFS with its native checksumming.

## Summary

FIM is a critical component of the "Detect" phase of the NIST Cybersecurity Framework. By using Monit to watch your most sensitive files, you gain immediate visibility into unauthorized system modifications, allowing you to respond before an attacker can solidify their foothold.
