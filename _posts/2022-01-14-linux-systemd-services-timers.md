---
layout: post
title: "Linux - Systemd Overrides: The Professional Way to Modify Package-Managed Services"
date: 2022-01-14 01:33:57 +01
categories: linux administration
tags: systemd unit-files administration linux best-practices
---

## The Problem: The Overwrite Trap

You need to change a service's environment variables or its OOM score. You edit `/lib/systemd/system/nginx.service` directly. A month later, `apt upgrade` runs, a new Nginx version is installed, and your customisations are wiped out. This is the most common mistake made by junior administrators.

## The Optimal Solution: Drop-in Overrides

Systemd provides a mechanism to "overlay" changes without touching the original unit file. These are called drop-in files, located in a directory named `<unit>.service.d/`.

### 1. The Easy Way: systemctl edit

The `edit` command handles the directory creation and the file naming for you.

```bash
sudo systemctl edit nginx.service
```

This opens your default editor. Anything you add here is merged with the base configuration.

### 2. Common Advanced Overrides

#### Changing the Restart Policy

Force a service to wait longer between restart attempts to prevent log flooding:

```ini
[Service]
RestartSec=30s
StartLimitIntervalSec=0
```

#### Mounting an Environment File

Safely inject secrets or configuration variables:

```ini
[Service]
EnvironmentFile=/etc/default/myapp-secrets
```

#### Adjusting Resource Limits

Prevent a memory-hungry service from crashing the whole host:

```ini
[Service]
MemoryMax=2G
CPUWeight=50
```

## Troubleshooting Key Considerations

- **The 'Empty Value' Reset**: If a parameter in the original unit file is a list (like `ExecStart=`), you must first provide an empty value to "clear" it before adding your new value.
  ```ini
  [Service]
  ExecStart=
  ExecStart=/usr/local/bin/custom-nginx -g 'daemon off;'
  ```
- **Checking the Result**: Always verify that your override was applied correctly using `systemctl show`:
  ```bash
  systemctl show nginx.service -p ExecStart
  ```
- **The systemd-delta tool**: Use `systemd-delta` to see all overrides and modifications across your entire system. It's a great way to audit a server you've just inherited.

## Summary

Drop-in files are the foundation of clean system administration. They allow for version-control-friendly configuration management (via Ansible/Puppet) while ensuring that package updates never break your custom system tuning.
