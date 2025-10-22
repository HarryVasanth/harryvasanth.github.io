---
layout: post
title: "Networking - Security: Implementing WPA3-Enterprise with RADIUS and Certificates"
date: 2025-10-22 14:49:18 +01
categories: networking security
tags: wpa3 radius networking security linux administration
---

## The Weakness of Shared Passwords (WPA2-PSK)

In an enterprise environment, using a single Wi-Fi password for 200 employees is a massive security risk. When an employee leaves, you must change the password for everyone. If one laptop is stolen, the attacker has the master key to your entire network.

**WPA3-Enterprise** (with 802.1X) solves this by providing **Unique, Per-User Authentication**. Instead of a password, each device uses its own digital certificate or individual credentials to talk to a **RADIUS** server. WPA3 also introduces **SAE (Simultaneous Authentication of Equals)**, which makes "Offline Dictionary Attacks" (cracking the password after capturing the handshake) mathematically impossible.

## The Architecture: The AAA Pipeline

1.  **Supplicant**: The user's device (Laptop/Phone).
2.  **Authenticator**: The Wi-Fi Access Point (Unifi, Aruba, or OPNsense).
3.  **Authentication Server**: **FreeRADIUS** running on Linux.
4.  **Identity Store**: **OpenLDAP** or **Active Directory**.

## Implementation: Configuring FreeRADIUS for EAP-TLS

EAP-TLS (using certificates) is the most secure form of Wi-Fi authentication. No passwords are ever sent or even exist.

### 1. Generate the Certificates

You need a Certificate Authority (CA), a Server Certificate for FreeRADIUS, and a Client Certificate for each user.

```bash
# Using the FreeRADIUS helper scripts
cd /etc/freeradius/3.0/certs/
./bootstrap
```

### 2. Define the Client (The Access Point)

FreeRADIUS must know the IP and the shared secret of your Wi-Fi Access Point.

```text
# /etc/freeradius/3.0/clients.conf
client ap-hallway {
    ipaddr = 10.0.1.5
    secret = my_ultra_secret_shared_key
}
```

### 3. Configure EAP

Ensure FreeRADIUS is set to use the certificates we generated.

```text
# /etc/freeradius/3.0/mods-enabled/eap
eap {
    default_eap_type = tls
    tls-config default {
        private_key_file = ${certdir}/server.key
        certificate_file = ${certdir}/server.pem
        ca_file = ${certdir}/ca.pem
    }
}
```

## Phase 2: Enforcing WPA3-Enterprise

On your Wi-Fi Controller (e.g., Unifi):

1.  **Security**: Select `WPA3-Enterprise`.
2.  **RADIUS Profile**: Enter the IP of your FreeRADIUS server and the shared secret.
3.  **PMF (Protected Management Frames)**: Set to `Required`. This is a mandatory part of WPA3 that prevents "Deauthentication Attacks" (where an attacker forcibly kicks users off the network).

## Why This is a Suitable Workflow

- **Role-Based Access**: Based on the user's certificate, the RADIUS server can tell the Access Point to put that specific user into a specific VLAN. (e.g., Marketing goes to VLAN 10, Finance goes to VLAN 20).
- **Instant Revocation**: If a device is lost, you simply revoke its certificate in the CA. The device can never connect again, while no other user is affected.
- **No Password Fatigue**: Users don't have to remember (and write down) complex Wi-Fi passwords.

## Summary

WPA3-Enterprise is the gold standard for corporate wireless security. By moving to a certificate-based RADIUS architecture, you eliminate the "Shared Password" liability and gain total control over who is on your network. It is a complex setup, but it is the only way to build a truly professional, high-security wireless infrastructure in 2025.
