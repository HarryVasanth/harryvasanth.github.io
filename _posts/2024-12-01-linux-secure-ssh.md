---
layout: post
title: "Linux - Secure SSH with Fail2ban, Key Authentication, and Advanced Security Practices"
date: 2024-12-01 11:32:07 +01
categories: linux
tags: security fail2ban hosting
---

## Intro

Securing SSH is critical to protect your server from unauthorized access and cyber attacks. This guide expands on basic SSH security by introducing advanced practices, including key authentication, Fail2ban configuration, and additional hardening techniques to fortify your server.

---

## Step 1: Disable Password Authentication and Root Login

Edit the SSH configuration file:

```bash
sudo nano /etc/ssh/sshd_config
```

Make the following changes:

```bash
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitUserEnvironment no
X11Forwarding no
AllowTcpForwarding no
LogLevel VERBOSE
MaxAuthTries 4
MaxSessions 4
```

- **Disable Root Login:** Prevent attackers from targeting the root user.
- **Disable Password Authentication:** Eliminates brute-force password attacks.
- **Disable X11 Forwarding:** Prevents graphical application exploitation.
- **Limit Authentication Attempts:** Reduces brute-force attack success rates.
- **Set Verbose Logging:** Enhances monitoring for suspicious activity.

Save and restart the SSH service:

```bash
sudo systemctl restart sshd
```

---

## Step 2: Set Up SSH Key Authentication

### Generate a Strong Key Pair

Use Ed25519 for better performance and security:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Alternatively, use RSA (4096 bits):

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

### Copy the Public Key to the Server

```bash
ssh-copy-id user@your_server_ip
```

Or manually add the public key to `~/.ssh/authorized_keys` on the server.

### Secure Your Private Key

- Use a strong passphrase when generating keys.
- Store private keys securely (e.g., encrypted storage).
- Avoid embedding keys in scripts or applications.

Test the connection:

```bash
ssh user@your_server_ip
```

---

## Step 3: Install and Configure Fail2ban

### Install Fail2ban

```bash
sudo apt update && sudo apt install fail2ban -y
```

### Configure Fail2ban Jail

Create or edit a custom jail configuration:

```bash
sudo nano /etc/fail2ban/jail.local
```

Add the following:

```bash
[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
maxretry = 5
findtime = 600
bantime = 3600

[recidive]
enabled = true
logpath = /var/log/fail2ban.log
bantime = 86400  # Ban for one day after repeated offenses.
findtime = 86400 # Time window to track repeated offenses.
maxretry = 3     # Number of retries before banning.
```

Restart Fail2ban:

```bash
sudo systemctl restart fail2ban
```

---

## Step 4: Advanced SSH Hardening Techniques

### Change Default SSH Port

To reduce automated attacks, change the default port (e.g., to `2222`):

```bash
sudo nano /etc/ssh/sshd_config
```

Add or modify:

```bash
Port 2222
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

**Note:** Update firewall rules to allow the new port.

### Restrict User Access

Limit SSH access to specific users or groups:

```bash
AllowUsers user1 user2@IP1 user3@IP2
AllowGroups sshusers
DenyUsers baduser
DenyGroups restrictedgroup
```

### Enable Two-Factor Authentication (2FA)

Install Google Authenticator for an additional layer of security:

```bash
sudo apt install libpam-google-authenticator -y
google-authenticator
```

Edit PAM configuration:

```bash
sudo nano /etc/pam.d/sshd
```

Add:

```bash
auth required pam_google_authenticator.so nullok
```

Update `/etc/ssh/sshd_config`:

```bash
ChallengeResponseAuthentication yes
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

---

## Step 5: Monitor and Audit SSH Activity

### Monitor Logs in Real-Time

Use `tail` to monitor authentication logs:

```bash
sudo tail -f /var/log/auth.log
```

### Audit Existing Keys

Identify unused or shadow keys with tools like `ssh-audit`.

---

## Step 6: Automate Key Management and Rotation

Use OpenSSH certificates for centralized key management.

#### Generate a CA Key:

```bash
ssh-keygen -t rsa -b 4096 -f /path/to/ca -C "CA for SSH"
```

#### Sign User Keys:

```bash
ssh-keygen -s /path/to/ca -I user_key_id -n username -V +52w user_key.pub
```

#### Configure Trusted CA:

Add this to `/etc/ssh/sshd_config`:

```bash
TrustedUserCAKeys /path/to/ca.pub
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

---

## Step 7: Verify Security Setup

### Test Fail2ban

Simulate failed logins and verify bans:

```bash
sudo fail2ban-client status sshd
```

### Check Firewall Rules

Ensure only necessary ports are open:

```bash
sudo ufw status verbose
```

---

## Conclusion

By implementing these advanced techniques, you significantly enhance your server's resilience against attacks. Regularly audit configurations, rotate keys, monitor logs, and stay updated with security patches. Remember, security is an ongoing process that requires vigilance.
