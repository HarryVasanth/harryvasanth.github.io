---
layout: post
title: "Linux - SELinux: A Deep Dive into Mandatory Access Control and Custom Policy Authoring"
date: 2022-12-28 13:37:38 +01
categories: linux security
tags: selinux security linux troubleshooting hardening administration
---

## The Security Administrator's Enigma: SELinux

For many Linux administrators, Security-Enhanced Linux (SELinux) is the first thing disabled after a fresh installation. This is a significant mistake. SELinux provides a mandatory access control (MAC) mechanism that can stop an attacker even if they have gained root privileges via a kernel exploit or a misconfigured application. Disabling it is equivalent to removing the internal fire doors in a building because they are "slightly inconvenient" to walk through.

In a professional environment, we cannot rely solely on Discretionary Access Control (DAC) - the standard owner/group permissions. DAC is too blunt; if a process runs as root, it can do anything. SELinux adds a second layer of defense that asks: "Even if this process is root, is it _allowed_ to perform this specific action on this specific file?"

## The Three Pillars of SELinux

To master SELinux, you must understand its core components: Contexts, Booleans, and Policies.

### 1. Security Contexts (The Labels)

Every process, file, and network socket in an SELinux-enabled system has a label. These labels follow the format `user:role:type:level`.

- **Type (`_t`)**: This is the most important part for administrators. It defines the "domain" of a process or the "type" of a file. For example, the Nginx process runs with the `httpd_t` type, and web files should have the `httpd_sys_content_t` type.
- **Verification**: You can see these labels using the `-Z` flag in standard commands:

```bash
ls -Z /var/www/html
ps -eZ | grep nginx
```

### 2. SELinux Booleans (The Toggles)

Booleans are on/off switches that allow you to change SELinux behavior at runtime without writing new policies.

- **Common Use Case**: Allowing a web server to send email.

```bash
# Check the status of a boolean
getsebool httpd_can_sendmail
# Enable it permanently
setsebool -P httpd_can_sendmail 1
```

## The Diagnostic Workflow: Solving Denials

When a service fails with a "Permission Denied" error, but the standard permissions look correct, SELinux is the primary suspect.

### Step 1: Auditing the Denials

SELinux denials are recorded in the audit log as "AVC" (Access Vector Cache) messages.

```bash
# Search for recent denials
ausearch -m AVC -ts recent
```

### Step 2: Human-Readable Analysis

The `sealert` tool (part of `setroubleshoot-server`) provides clear explanations and suggested fixes.

```bash
sealert -a /var/log/audit/audit.log
```

## Strategy: Correcting Context Drift

The most frequent cause of SELinux issues is moving files (e.g., using `mv`) instead of copying them. Moving a file preserves its original context (like `user_home_t`), which the web server (`httpd_t`) is not allowed to read.

**The Fix**: Use `restorecon` to reset the context based on the system's global policy database.

```bash
# Recursively restore the default contexts
restorecon -Rv /var/www/html
```

## Advanced: Creating Custom Policy Modules

Sometimes, you are running a custom application that performs legitimate actions not covered by the default policy. In this case, you should create a custom policy module rather than setting the whole system to `permissive` mode.

1.  **Capture the error**: Put the system in permissive mode temporarily and run the app.

```bash
setenforce 0
# Run the application...
setenforce 1
```

2.  **Generate the policy**: Use `audit2allow` to translate the audit logs into a policy.

```bash
ausearch -m AVC -ts recent | audit2allow -M my_custom_app
```

3.  **Load the policy**:

```bash
semodule -i my_custom_app.pp
```

## The 'No-Disable' Manifesto

A experienced administrator treats SELinux as a partner, not an obstacle. By mastering context management and the `audit2allow` workflow, you maintain a high-security posture while ensuring your services remain functional. Disabling SELinux is a sign of operational surrender; fixing the policy is a sign of technical mastery.

## Best Practices for Production

1.  **Never Use Permissive Mode as a Permanent Fix**: It is a diagnostic tool only.
2.  **Labeling at the Source**: When using automation like Ansible or Terraform, always specify the SELinux context in the file/template resource.
3.  **Monitor the Audit Log**: Use a tool like Logcheck or a central SIEM to alert you to unexpected AVC denials.
4.  **Keep it Simple**: Before writing a custom policy, check if a Boolean already exists for your requirement.

## Summary

SELinux is the definitive security layer for modern Linux servers. It provides the isolation needed to run complex, exposed services with confidence. By understanding the relationship between process types and file types, and by leveraging the powerful diagnostic tools available, you transform a complex system into a manageable, bulletproof defense. This rigour is what defines a production-ready infrastructure environment.
