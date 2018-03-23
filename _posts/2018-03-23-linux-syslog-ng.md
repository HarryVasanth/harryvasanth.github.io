---
layout: post
title: "Linux - Syslog-ng"
date: 2018-03-23 18:29:11 +01
categories: linux logs
tags: syslog-ng logs
---

## Introduction

Syslog-ng is a powerful log management tool that allows flexible and centralized logging.

## Installation

Ensure your system is up-to-date and install syslog-ng using your package manager:

```bash
sudo apt update
sudo apt install syslog-ng
```

## Basic Configuration

Syslog-ng's configuration file is typically located at `/etc/syslog-ng/syslog-ng.conf`. Let's start with a simple configuration:

```conf
@version: 3.32
@include "scl.conf"

options {
    flush_lines (0);
    time_reopen (10);
    log_fifo_size (1000);
    long_hostnames (off);
    use_dns (no);
    use_fqdn (no);
    create_dirs (no);
    keep_hostname (yes);
};

source s_net { tcp(); udp(); };

destination d_messages { file("/var/log/messages"); };
log { source(s_net); destination(d_messages); };
```

This configuration sets up syslog-ng to listen for log messages over TCP and UDP, then writes them to a file at `/var/log/messages`.

## Example Configurations and Use Cases

### 1. Separate Application Logs

You can separate logs from different applications:

```conf
source s_app { udp(port(514) flags(no-hostname)); };
destination d_app { file("/var/log/app.log"); };
log { source(s_app); destination(d_app); };
```

### 2. Centralized Logging

Syslog-ng allows you to centralize logs from multiple sources:

```conf
source s_network {
    tcp(ip("192.168.1.1") port(514));
    tcp(ip("192.168.1.2") port(514));
};
destination d_central { file("/var/log/central.log"); };
log { source(s_network); destination(d_central); };
```

## Security Considerations

### 1. Encrypting Log Messages

To encrypt log messages, you can use TLS:

```conf
source s_secure { tcp(port(6514) tls(peer-verify(required-untrusted)) ); };
```

### 2. Rate Limiting

Prevent log flooding with rate-limiting:

```conf
source s_ratelimit { tcp(port(514) flags(no-hostname) flags(no-parse)); };
destination d_ratelimit { usertty("root") file("/var/log/ratelimit.log" perm(0640)); };
log { source(s_ratelimit); destination(d_ratelimit); };
```

## Advanced Features

### 1. Parsing JSON Logs

Syslog-ng can parse structured logs, such as JSON:

```conf
parser p_json {
    json-parser(prefix(".json."));
};
source s_json { tcp(port(514) parser(p_json)); };
destination d_json { file("/var/log/json.log"); };
log { source(s_json); destination(d_json); };
```

### 2. Executing External Commands

Execute external commands based on log content:

```conf
filter f_error { level(err..emerg); };
destination d_exec { program("/path/to/script.sh"); };
log { source(s_network); filter(f_error); destination(d_exec); };
```

## Optimization

### 1. Buffering

Optimize syslog-ng's performance by buffering logs:

```conf
log { source(s_network); destination(d_central); };
options { chain_hostnames(off); };
```

### 2. Disk Buffering

Use disk buffering for increased reliability:

```conf
destination d_diskbuffer {
    file("/var/log/diskbuffer"
    template("$(format-json --scope rfc5424 --scope nv-pairs --exclude DATE --key ISODATE)\n")
    template_escape(no)
    template-escape(no)
    disk-buffer( persist-name("diskbuffer") max-size(100M) );
};
log { source(s_network); destination(d_diskbuffer); };
```

## Conclusion

Syslog-ng is a versatile logging tool that provides extensive customization options. Tailor the configurations to your specific needs for an effective and efficient log management system.
