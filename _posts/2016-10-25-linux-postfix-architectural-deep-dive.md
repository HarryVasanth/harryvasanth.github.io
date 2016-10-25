---
layout: post
title: "Linux - Postfix: Advanced Domain Class Architecture and Message Rewriting"
date: 2016-10-25 18:10:21 +01
categories: linux mail
tags: postfix mail-server architecture networking linux troubleshooting
---

## Introduction to the Postfix Message Pipeline

Postfix is not a single, monolithic application. It is a sophisticated ecosystem of specialised daemons, each designed to handle a specific stage of the mail delivery process. This modular architecture is the key to its reputation for security and performance. At the heart of this system is the `trivial-rewrite` daemon. This component is responsible for the fundamental task of determining how a message should be routed based on its destination domain. For an infrastructure administrator, mastering the classification of these domains is critical for maintaining a reliable mail system.

Every domain that Postfix interacts with must fall into exactly one of four logical categories. If a domain is misclassified or, worse, listed in multiple categories, Postfix encounters logical ambiguities that result in delivery failures, typically manifesting as "Relay Access Denied" or "User Unknown" errors. Understanding the precedence and purpose of these four classes is a prerequisite for any professional mail server deployment.

## The Four Mandatory Domain Classes

### 1. The Local Domain Class (`mydestination`)

The Local class represents the server's own identity. For a domain in this class, Postfix assumes that the recipients are actual physical accounts on the Linux system.

- **Mechanism**: The `local` delivery agent handles these messages. It looks for users in `/etc/passwd` or system-level aliases in `/etc/aliases`.
- **Best Practice**: In modern environments where we typically use virtual mailboxes, the `mydestination` parameter should be kept extremely minimal. Usually, only `localhost`, `localhost.$mydomain`, and the server's fully qualified hostname should be present here.

### 2. The Virtual Alias Class (`virtual_alias_domains`)

This class is used primarily for forwarding and aliasing. A domain in this class does not have any physical or even virtual mailboxes of its own.

- **Mechanism**: The `cleanup` daemon performs a lookup in the `virtual_alias_maps`. Every address in this domain must be mapped to a destination in a different domain.
- **Usage**: Excellent for "catch-all" domains or for vanity domains that simply forward mail to a primary corporate account.

### 3. The Virtual Mailbox Class (`virtual_mailbox_domains`)

This is the industry standard for modern, multi-domain mail servers. The users in this class do not exist in the system's `/etc/passwd` file.

- **Mechanism**: User data is typically stored in a relational database like MariaDB or an LDAP directory. Delivery is handled by the `virtual` agent or, in more advanced setups, handed off via the Local Mail Transfer Protocol (LMTP) to a specialised IMAP server such as Dovecot.
- **Benefit**: This allows for thousands of mailboxes without cluttering the host operating system with system users.

### 4. The Relay Domain Class (`relay_domains`)

This class is for domains where the server acts as an intermediary gateway or a backup MX.

- **Mechanism**: Postfix accepts the mail and immediately queues it for forwarding to a downstream server (the "relay host"). It does not attempt local delivery or virtual lookups.

## The Precedence Trap: Why Deliveries Fail

A common architectural failure occurs when a domain is accidentally defined in multiple classes-most frequently in both `mydestination` and `virtual_mailbox_domains`.

When the `smtpd` daemon receives a message, it queries the `trivial-rewrite` daemon to find the domain's class. Postfix checks these classes in a hardcoded order of precedence. If a domain is listed in `mydestination`, Postfix treats it as a Local domain. If you are actually using Dovecot (a Virtual Mailbox setup) for that domain, Postfix will look for the user in the system's `/etc/passwd` file. When it fails to find a system user named 'john@example.com', it will bounce the mail with a "550 User unknown in local recipient table" error. It never even reaches the stage of checking your virtual database.

### Technical Resolution Workflow

To resolve such issues, you must audit the current classification and ensure strict isolation.

```bash
# Check the current values of the two most common conflicting parameters
postconf mydestination virtual_mailbox_domains
```

If your primary mail domain appears in both outputs, you must remove it from the Local class. A professional `/etc/postfix/main.cf` should look like this:

```conf
# /etc/postfix/main.cf

# ONLY local system addresses and localhost
mydestination = $myhostname, localhost.$mydomain, localhost

# Your hosted mail domains are defined here
virtual_mailbox_domains = hash:/etc/postfix/vmail_domains
```

{: file='/etc/postfix/main.cf'}

## Deep Dive: Address Rewriting and Masquerading

Beyond simple classification, Postfix provides a powerful set of tools for modifying addresses as they traverse the pipeline. This is particularly useful in enterprise settings where internal servers might send mail as `root@app-server-01.internal.local`, but you want the external world to see a professional address like `alerts@example.com`.

### Canonical Maps

Canonical mapping allows you to change both the envelope address (used for routing) and the header address (what the user sees in their mail client). This is a deep transformation that happens early in the `cleanup` phase.

```conf
sender_canonical_maps = hash:/etc/postfix/sender_canonical
```

Example `/etc/postfix/sender_canonical` content:

```text
root@app-server-01.internal.local    admin@example.com
```

### Generic Maps

If you only need to change addresses for mail leaving the system via SMTP (e.g., when relaying through an external ISP), use `smtp_generic_maps`. This is safer as it doesn't affect local delivery logic or internal routing.

## Performance Tuning: The Proxymap Daemon

On high-volume nodes, the overhead of establishing new database connections (MySQL/LDAP) for every Postfix worker process can become a significant bottleneck. Postfix provides the `proxymap` daemon to mitigate this. By prefixing your lookup tables with `proxy:`, you tell the worker processes to share a single connection pool maintained by the proxy daemon.

```conf
# Use the proxy daemon for database queries to improve performance
virtual_mailbox_maps = proxy:mysql:/etc/postfix/mysql_virtual_users.cf
```

## Advanced Security: SMTP Smuggling Mitigation

In late 2023, a significant protocol-level vulnerability known as "SMTP Smuggling" was identified. This exploit relies on discrepancies in how different mail servers handle non-standard line endings. An attacker can "smuggle" a second, separate email inside the data portion of a first email, bypassing SPF and DKIM checks.

As an infrastructure administrator, you must ensure your Postfix version is up to date and configured to strictly enforce line-ending standards.

```conf
# SMTP Smuggling Hardening
smtpd_forbid_bare_newline = yes
smtpd_forbid_bare_newline_exclusions = $mynetworks
```

## Debugging with 'postmap' and 'smtpd -v'

When address rewriting isn't behaving as expected, you can test the lookup logic manually without sending emails:

```bash
# Test a virtual alias lookup
postmap -q "alias@example.com" hash:/etc/postfix/virtual
```

For a deeper look, you can increase the verbosity of a specific daemon in `master.cf`:

```conf
# master.cf
smtp      inet  n       -       y       -       -       smtpd -v
```

## Summary

Postfix is a masterpiece of modular engineering, but that modularity requires the administrator to be explicit. By strictly separating your domain classes and understanding the message rewriting pipeline, you build a mail infrastructure that is both predictable and scalable. Always verify your changes using `postmap -q` before reloading the service, and keep a close watch on `/var/log/mail.log` for the `relay=local` vs `relay=virtual` indicators during your testing phase. This level of rigour is what defines a experienced system administrator's approach to critical infrastructure.
