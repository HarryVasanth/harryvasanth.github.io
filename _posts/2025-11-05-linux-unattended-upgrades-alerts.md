---
layout: post
title: "Linux - Maintenance: Advanced Unattended-Upgrades with Slack/Teams Alerts"
date: 2025-11-05 04:24:12 +01
categories: linux administration
tags: linux maintenance automation security administration troubleshooting
---

## The 'Manual Update' Bottleneck

If you are managing more than five Linux servers, manually running `apt update && apt upgrade` is a waste of your professional time. More importantly, it is a security risk-vulnerabilities are often exploited within hours of a patch being released. You cannot afford to wait until "Maintenance Monday" to patch a zero-day in OpenSSL.

**Unattended-Upgrades** is the standard tool for automated patching on Debian/Ubuntu. However, a experienced administrator knows that "blind" updates are dangerous. You need to know exactly what was updated, if anything failed, and if a reboot is required.

## Phase 1: Hardening the Configuration

We want to automate security patches but perhaps keep "App" updates manual to avoid breaking things.

```text
# /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    // Optional: Allow stable updates too
    // "${distro_id}:${distro_codename}-updates";
};

# Never update these critical packages automatically
Unattended-Upgrade::Package-Blacklist {
    "mysql-server";
    "postgresql-*";
};

# Automatically reboot at 03:00 AM if required by a kernel update
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

## Phase 2: Implementing Real-Time Alerts (The Ideal Fix)

The default `unattended-upgrades` can send emails, but no one reads server emails. We want a notification in our Slack or Microsoft Teams "Ops" channel.

We use a "Post-Invoke" script in APT to trigger a webhook.

```bash
#!/bin/bash
# /usr/local/bin/notify-apt.sh

# Get the last few lines of the history log
LOG_DATA=$(tail -n 20 /var/log/apt/history.log)

# Format the JSON for Slack
PAYLOAD=$(jq -n --arg msg "Updates applied on $(hostname):\n\$LOG_DATA" '{text: \$msg}')

# Send to the Webhook
curl -X POST -H 'Content-type: application/json' --data "$PAYLOAD" https://hooks.slack.com/services/XXXX/YYYY/ZZZZ
```

### Linking to APT

Create `/etc/apt/apt.conf.d/99-notifications`:

```text
DPkg::Post-Invoke { "/usr/local/bin/notify-apt.sh"; };
```

## Phase 3: Handling 'Pending Reboot' Visibility

One of the biggest issues with automated updates is a server that has been patched but is still running an old, vulnerable kernel because it hasn't been rebooted.

**The Advanced Tool**: `needrestart`.
`needrestart` checks which processes are using old versions of libraries and whether the kernel is outdated.

```bash
sudo apt install needrestart
# Check the status
sudo needrestart -b
```

## Why This is a Suitable Workflow

- **Security Compliance**: You can prove to auditors that security patches are applied within 24 hours of release.
- **Operational Awareness**: Your whole team sees the Slack alert. "Ah, the staging server just rebooted for a kernel patch, that explains the 30-second blip in the metrics."
- **Risk Mitigation**: By blacklisting sensitive databases, you ensure that complex stateful migrations only happen under human supervision.

## Summary

Automated maintenance is a prerequisite for scaling. By combining `unattended-upgrades` with granular filters and real-time webhook notifications, you achieve the perfect balance: high security through rapid patching, and high visibility through automated reporting. This is how you manage a fleet of 100 servers with the same effort as a fleet of one.
