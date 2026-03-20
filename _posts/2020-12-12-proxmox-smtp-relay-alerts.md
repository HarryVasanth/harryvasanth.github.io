---
layout: post
title: "Proxmox - Networking: Configuring SMTP Relay for Cluster-Wide Alerts"
date: 2020-12-12 19:06:54 +01
categories: proxmox administration
tags: proxmox mail postfix alerts administration notification
---

## The Criticality of Infrastructure Visibility

In a production Proxmox environment, you cannot afford to be blind to the health of your nodes. Whether it is a ZFS disk failure, a backup error, or a high-temperature alert from a CPU, the hypervisor must have a reliable way to reach you. By default, Proxmox attempts to send emails using its local `postfix` instance. However, in the modern era of strict SPF, DKIM, and DMARC policies, mail sent directly from a residential or datacenter IP is almost always blocked or marked as spam.

To ensure you never miss a critical alert, you **must** configure Proxmox to use a professional SMTP relay (like SendGrid, Mailgun, or your own corporate SMTP server).

## Phase 1: Installing the Authentication Requirements

Postfix needs specific libraries to handle SASL (Simple Authentication and Security Layer) encryption, which is required by almost all relay providers.

```bash
apt update && apt install libsasl2-modules
```

## Phase 2: Configuring Postfix for the Relay

Edit the main Postfix configuration file: `/etc/postfix/main.cf`. You should add the following parameters to the end of the file.

```conf
# The address of your relay provider (e.g., SendGrid)
relayhost = [smtp.sendgrid.net]:587

# Enable SASL authentication
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous

# Enforce TLS encryption
smtp_use_tls = yes
smtp_tls_security_level = encrypt
smtp_tls_note_starttls_offer = yes
```

## Phase 3: Storing the Credentials Securely

Create the password file: `/etc/postfix/sasl_passwd`.

```text
[smtp.sendgrid.net]:587    apikey:YOUR_ACTUAL_API_KEY_HERE
```

**CRITICAL: Security Fix**
Because this file contains a plain-text API key, you must restrict its permissions immediately.

```bash
chmod 600 /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd
```

## Phase 4: Setting the 'From' Address

Proxmox often sends mail as `root@pve.localdomain`. Many relays will reject this because it doesn't match a real domain. We must use Postfix's "Generic" mapping to rewrite the sender address.

1.  Edit `/etc/postfix/generic`:

```text
root@pve.localdomain    alerts@yourdomain.com
root@pve               alerts@yourdomain.com
```

2.  Apply the map in `main.cf`:

```conf
smtp_generic_maps = hash:/etc/postfix/generic
```

3.  Update the map and reload:

```bash
postmap /etc/postfix/generic
systemctl reload postfix
```

## Phase 5: Testing the Alert Pipeline

Don't wait for a real disk failure to test your alerts. Use the `mail` utility to send a test message.

```bash
echo "This is a test alert from Proxmox Cluster A" | mail -s "Proxmox Test" your-email@example.com
```

Check `/var/log/mail.log` to verify that the message was accepted by the relay and not rejected.

## Suitable Strategy: Centralised Alerting

In a large cluster, instead of configuring every node individually, a experienced administrator uses an automation tool (like Ansible) to push these configurations. This ensures that every new node added to the datacenter is automatically integrated into the global alerting fabric.

## Summary

An infrastructure that doesn't alert is an infrastructure that is already failing. By implementing a robust SMTP relay with proper SASL authentication and address rewriting, you ensure that Proxmox's "cries for help" actually reach your inbox. This level of operational maturity is a prerequisite for any administrator who values system uptime and peace of mind in 2020.
