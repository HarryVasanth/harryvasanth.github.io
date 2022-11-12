---
layout: post
title: "Networking - OPNsense: Implementing GeoIP and CrowdSec for Proactive Edge Defense"
date: 2022-11-12 01:16:07 +01
categories: networking security
tags: opnsense crowdsec firewall security networking hardening
---

## The Threat: Automated Opportunistic Attacks

If you check your firewall logs, you'll see thousands of attempts to hit SSH, RDP, or common web vulnerabilities (like `/wp-admin`) every hour. These aren't targeted; they are automated bots scanning the entire internet.

## Strategy 1: GeoIP 'Zero-Trust' Blocking

If your business only operates in the UK, there is no reason to allow traffic from the rest of the world to hit your management ports.

### Implementation (OPNsense)

1.  **Register for MaxMind**: Get a free license key for their GeoLite2 database.
2.  **Firewall -> Aliases -> GeoIP**: OPNsense will automatically download the IP ranges for each country.
3.  **Create an Alias**: Name it `Allowed_Countries`.
4.  **Firewall Rule**: Create a "Block" rule on your WAN interface where Source is NOT `Allowed_Countries`.

## Strategy 2: CrowdSec (The Modern Fail2Ban)

CrowdSec is a collaborative IDS. When one user's OPNsense detects an attack, the IP is shared with the entire "crowd," and everyone else's firewall automatically blocks it.

### Implementation

1.  **Install the Plugin**: `os-crowdsec` via the OPNsense plugin manager.
2.  **Configure Scenarios**: Enable scenarios for SSH brute-force, HTTP probing, and port scanning.
3.  **Install the Bouncer**: The "Bouncer" is what actually talks to the OPNsense firewall (pf) to drop the packets.

## Tips & Tricks

- **The 'False Positive' Safety Net**: Never block "All traffic" via GeoIP for ports that need to be public (like SMTP or a public website). You will break legitimate traffic from CDNs, search engine crawlers (Googlebot), and users on VPNs.
- **Floating Rules**: In OPNsense, use "Floating Rules" for your GeoIP blocks. This allows you to apply the block to all interfaces (WAN, DMZ, VPN) simultaneously, ensuring an attacker can't move laterally if they manage to get into a lower-security segment.
- **Monitoring**: Check the CrowdSec dashboard (`cscli alerts list`) to see which IPs are being blocked. A well-versed admin looks for patterns: are all the attacks coming from one specific ISP or Hosting provider?

## Summary

Traditional firewalls are reactive. By combining GeoIP (limiting the attack surface) with CrowdSec (collaborative intelligence), you create a proactive edge defence that stops attackers before they even make their first request to your servers.
