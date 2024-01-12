---
layout: post
title: "Linux - RSyslog"
date: 2018-04-18 17:21:03 +01
categories: linux
tags: rsyslog logs
---

## Introduction

RSyslog is a robust and flexible syslogd replacement, providing advanced features for logging on Unix-like systems. In this comprehensive guide, we will cover the installation, basic configurations, security considerations, advanced features, and optimization techniques for RSyslog.

### Installation

To install RSyslog on your system, use your distribution's package manager. For example, on Ubuntu:

```bash
sudo apt update
sudo apt install rsyslog
```

For other distributions, you can use tools like `yum` or `zypper`.

## Basic Configuration

RSyslog configuration files are usually located in `/etc/rsyslog.conf` and `/etc/rsyslog.d/`. Here is a basic configuration to get you started:

```bash
# /etc/rsyslog.conf

# Enable UDP syslog reception
$ModLoad imudp
$UDPServerRun 514

# Log messages from local0 facility to a separate file
local0.* /var/log/local0.log
```

This basic setup enables UDP syslog reception on port 514 and directs messages from the local0 facility to a dedicated log file.

## Use Cases

### Centralized Logging

One common use case is setting up RSyslog as a centralized logging server. Clients across the network can forward their logs to the central server for aggregation and analysis.

#### Configuration on the Client

```bash
# /etc/rsyslog.conf

*.* @central_log_server_ip:514
```

### Log Rotation

RSyslog integrates with log rotation tools like `logrotate` to manage log files. Ensure log rotation is configured to avoid filling up the disk.

```bash
# /etc/logrotate.conf

/var/log/local0.log {
    rotate 7
    daily
    compress
    missingok
    notifempty
}
```

## Security Considerations

### Encryption

If transmitting logs over an insecure network, consider encrypting the communication. Use TLS for secure log forwarding:

```bash
# /etc/rsyslog.conf

$DefaultNetstreamDriver gtls
$DefaultNetstreamDriverCAFile /etc/ssl/certs/ca-certificates.crt
$ActionSendStreamDriverMode 1
$ActionSendStreamDriverAuthMode x509/name
$ActionSendStreamDriverPermittedPeer *.example.com
*.* @@(o)example.com:10514
```

### Access Control

Implement access control to restrict who can send logs to the RSyslog server:

```bash
# /etc/rsyslog.conf

$AllowedSender UDP, 127.0.0.1, 192.168.1.0/24
```

## Advanced Features

### Filtering

RSyslog supports powerful message filtering rules. For example, log only messages with a specific severity:

```bash
# /etc/rsyslog.conf

if $syslogseverity <= 4 then /var/log/severity_less_than_or_equal_to_4.log
& ~
```

### Templates

Customize log message format using templates:

```bash
# /etc/rsyslog.conf

$template CustomFormat,"[%msg%]"
*.* /var/log/custom_format.log;CustomFormat
```

## Optimization

### Batch Processing

To optimize performance, consider batching log entries before writing to disk:

```bash
# /etc/rsyslog.conf

$MainMsgQueueSize 100000
$MainMsgQueueDequeueSlowdown 0
$MainMsgQueueDiscardMark 97500
$MainMsgQueueHighWaterMark 80000
$MainMsgQueueLowWaterMark 60000
```

## Conclusion

RSyslog provides a powerful logging solution with extensive customization and security features.
