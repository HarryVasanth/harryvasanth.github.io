---
layout: post
title: "Linux - Security: Implementing ModSecurity WAF for Deep Layer 7 Protection"
date: 2020-04-20 19:15:25 +01
categories: linux security
tags: modsecurity security waf nginx ddos hardening linux web-server
---

## The Evolution of the DDoS Attack

Modern DDoS attacks have shifted up the OSI stack. While volumetric NTP or DNS amplification attacks still exist, they are easily mitigated by ISPs at the network edge. The more dangerous threat for a web administrator is the **Layer 7 (Application) Attack**. An attacker can use a small botnet to send a few hundred requests per second to a resource-intensive endpoint (like a search query or a PDF generator). This can consume 100% of your database CPU and RAM while your network traffic looks perfectly normal.

Traditional IP-based rate limiting at the firewall (iptables) is too blunt for this. We need a tool that can inspect the HTTP context and make stateful decisions: **ModSecurity**.

## Persistent Storage with ModSecurity

To perform effective rate limiting over time, ModSecurity needs to store data about IP addresses in a persistent memory zone. We use the `SecStorageEngine` and the `initcol` action to manage these collections.

### Implementation: Protecting a Heavy Search Endpoint

Imagine your site has a `/api/search` endpoint. We want to allow 10 searches per minute. If a user exceeds this, they get a 429 Too Many Requests for the next 10 minutes.

```conf
# 1. Initialize the IP collection
SecAction "id:9001,phase:1,nolog,pass,initcol:IP=%{REMOTE_ADDR}"

# 2. Track search requests
# We increment a variable and set its expiry to 60 seconds
SecRule REQUEST_URI "@beginsWith /api/search" \
    "id:9002,phase:1,nolog,pass,setvar:IP.search_count=+1,expirevar:IP.search_count=60"

# 3. Check for abuse
# If the count hits 10, we set a 'is_banned' flag for 10 minutes
SecRule IP:search_count "@gt 10" \
    "id:9003,phase:1,setvar:IP.is_banned=1,expirevar:IP.is_banned=600"

# 4. Enforce the ban
SecRule IP:is_banned "@eq 1" \
    "id:9004,phase:1,deny,status:429,msg:'Client temporarily banned for abuse',log"
```

{: file='/etc/nginx/modsec/main.conf'}

## The 'Real IP' Infrastructure Challenge

If your web server is behind a proxy or CDN (Cloudflare, AWS ALB, Nginx Ingress), `REMOTE_ADDR` will be the internal IP of the load balancer. If you apply the rules above without tuning, you will accidentally ban your own load balancer, taking your entire site offline for everyone.

### The Professional Fix: The Nginx real_ip Module

You must configure Nginx to extract the real client IP from headers like `X-Forwarded-For`.

```conf
# nginx.conf
set_real_ip_from 10.0.0.0/8;  # Your VPC range
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Once this is set, ModSecurity will see the actual attacker's IP address and apply the rate limit correctly.

## Handling False Positives with OWASP CRS

When you enable the OWASP Core Rule Set (CRS), you will inevitably block legitimate traffic. A common example is a CMS like WordPress, where a user submitting a long article might trigger SQL Injection or XSS rules because of the characters used in the post body.

**The Professional Tuning Workflow**:

1.  **Audit Mode**: Run `SecRuleEngine DetectionOnly` for 48-72 hours in production.
2.  **Analyse**: Search the audit log (`/var/log/modsec_audit.log`) for `msg` and `id` tags to find which rules are being triggered by real users.
3.  **Rule Exclusion**: Do not disable the rule globally. Instead, create a targeted exclusion for the specific URI and parameter.

```conf
# /etc/nginx/modsec/exclusions.conf
# Exclude Rule 942100 (SQLi) specifically for the blog body field
SecRuleUpdateTargetById 942100 "!REQUEST_BODY:content"
```

## Performance Impact and Rule Selection

Every rule in ModSecurity adds a few microseconds of latency to every request.

- **Rule Selection**: Only enable the rule groups you actually need.
- **Offloading**: For extremely high-traffic environments, consider offloading simple rate-limiting to a tool like **CrowdSec** or a CDN edge, using ModSecurity only for deep inspection.

## Summary

Layer 7 security is a continuous process of observation and tuning. By leveraging ModSecurity's persistent storage and granular rule language, you can protect your infrastructure from sophisticated, low-volume attacks that would bypass traditional defences. It is an essential component of a "Defense in Depth" strategy for any production web environment. A professional WAF implementation is the difference between an available service and a database under siege. In 2020, this is the gold standard for web server hardening.
