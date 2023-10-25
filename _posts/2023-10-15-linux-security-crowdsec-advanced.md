---
layout: post
title: "Linux - Security: Implementing CrowdSec for Collaborative Fleet-Wide Defense"
date: 2023-10-25 09:48:56 +01
categories: linux security
tags: crowdsec security linux fail2ban networking intrusion-prevention
---

## The Limitation of 'Isolated' Defense (Fail2Ban)

For years, `fail2ban` has been the standard for stopping brute-force attacks. It's simple: it reads your logs, finds a bad IP, and blocks it in `iptables`. However, `fail2ban` is reactive and isolated. It only knows about an attacker _after_ they have already tried to hit your specific server. In 2023, where botnets rotate thousands of IPs, this "local-only" knowledge is no longer sufficient for a professional production environment.

**CrowdSec** is the "modern, collaborative" successor to fail2ban. It uses a local agent to detect attacks, but it also shares those findings (anonymously) with a global community. If an IP is identified as a brute-forcer on a thousand other servers, your CrowdSec agent will block it **before** it even touches your firewall.

## The CrowdSec Architecture: Agent and Bouncer

To implement CrowdSec correctly, you must distinguish between its two components:

1.  **The Agent**: This is the "brain." It parses logs (using Grok patterns) and identifies malicious behavior (Scenarios).
2.  **The Bouncer**: This is the "muscle." It takes the decisions made by the agent and executes them at the Firewall (nftables), the Web Server (Nginx), or even the CDN (Cloudflare).

## Implementation: Protecting an Nginx Web Server

### 1. Installation

CrowdSec is written in Go and is extremely lightweight compared to the Python-based fail2ban.

```bash
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash
sudo apt install crowdsec
```

### 2. Installing Collections

Collections are pre-made sets of parsers and scenarios for specific services.

```bash
sudo cscli collections install crowdsecurity/nginx
sudo cscli collections install crowdsecurity/sshd
sudo systemctl reload crowdsec
```

### 3. Adding the Bouncer

For a general Linux server, the "Firewall Bouncer" using `nftables` is the best choice.

```bash
sudo apt install crowdsec-firewall-bouncer-nftables
```

## Suitable Strategy: The Local API (LAPI) and Multi-Server Setup

In a cluster, you don't want 10 servers all making their own decisions. You want a single **Local API** server that centralises the blocklist.

1.  **Master Node**: Runs the LAPI service.
2.  **Worker Nodes**: Run only the Agent and Bouncer, reporting back to the Master.
3.  **The Benefit**: If a bot hits Server A in your London datacenter, it is instantly blocked on Server B, C, and D in Tokyo and New York. This is "Fleet-Wide Immunity."

## Visualizing the Battle: The Dashboard and CLI

CrowdSec provides a web-based console and a powerful CLI (`cscli`) that allows you to see exactly who is hitting your infrastructure.

- **AS Reputation**: See if the attacks are coming from a specific cloud provider or country.
- **Attack Type**: Are they probing for `.env` files, brute-forcing SSH, or trying SQL Injection?
- **Metabase**: You can install a Metabase dashboard locally to get deep analytics of your blocked traffic over time.

## Performance: Why Go Matters for Security

Because CrowdSec is written in Go, its memory usage is predictable (typically < 100MB) regardless of how many logs it is parsing. Fail2Ban can struggle and consume massive CPU on high-volume logs due to its Python overhead and linear regex matching. CrowdSec uses an efficient event pipeline that handles thousands of log lines per second with minimal impact on application performance.

## Summary

Security is increasingly a game of data and collaboration. By moving from a "reactive/local" model to a "proactive/collaborative" model with CrowdSec, you significantly increase the cost for an attacker. You are no longer defending in a vacuum; you are part of a global immune system for the internet. This is a mandatory upgrade for any production Linux environment in the modern threat landscape. It represents the transition from "Individual Servers" to "Enterprise Fleet Protection."
