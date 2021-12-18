---
layout: post
title: "DevOps - Prometheus: Proactive SSL Monitoring with Blackbox Exporter"
date: 2021-12-18 11:10:16 +01
categories: devops monitoring
tags: prometheus ssl metrics security devops
---

## The Invisible Failure: Expired Certificates

Every well-versed admin has a horror story about a production outage caused by a forgotten SSL certificate. Modern infrastructure has hundreds of certificates (Internal PKI, Let's Encrypt, Cloudflare). Tracking them in a spreadsheet is a path to failure. You need an automated system that alerts you weeks before a crash happens.

## The Tool: Blackbox Exporter

The Prometheus **Blackbox Exporter** allows you to probe endpoints from the "outside." It doesn't just check if the server is up; it performs the full TLS handshake and extracts the certificate metadata.

### 1. Configuration

In your `blackbox.yml`:

```yaml
modules:
  http_2xx_tls:
    prober: http
    http:
      preferred_chain_brands: ["ISRG Root X1"] # Example for Let's Encrypt
      fail_if_ssl: false
      fail_if_not_ssl: true
```

### 2. Prometheus Scrape Job

Tell Prometheus which endpoints to check:

```yaml
scrape_configs:
  - job_name: "ssl_expiry"
    metrics_path: /probe
    params:
      module: [http_2xx_tls]
    static_configs:
      - targets:
          - https://example.com
          - https://api.mysite.net:443
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 127.0.0.1:9115 # Blackbox exporter address
```

{: file='prometheus.yml'}

## The Alert: The 'Golden Signal' for SSL

The metric you care about is `probe_ssl_earliest_cert_expiry`. It returns the expiry time as a Unix timestamp.

```yaml
groups:
  - name: SSL_Alerts
    rules:
      - alert: SSLCertExpiringSoon
        expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "SSL Certificate for {{ $labels.instance }} expires in {{ $value | humanizeDuration }}"
```

## Tips & Tricks

- **SNI Issues**: If you host multiple sites on one IP, ensure the Blackbox prober is sending the correct `server_name` (SNI). In newer versions, this is handled automatically via the `target` parameter.
- **Internal PKI**: To monitor internal services using a private CA, you must mount your CA certificate into the Blackbox Exporter's container at `/etc/ssl/certs`.
- **The 'Chain of Trust' Check**: `probe_ssl_last_chain_info` is a niche metric that tells you if the full chain (including intermediates) is being sent correctly. A browser might work with a missing intermediate, but many API clients (like Python's `requests`) will fail.

## Summary

Proactive monitoring is the hallmark of a experienced administrator. By integrating SSL expiry into your Prometheus/Alertmanager stack, you move from "Firefighting" to "Fire Prevention."
