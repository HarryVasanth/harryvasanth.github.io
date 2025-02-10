---
layout: post
title: "Linux - Configure Advanced Firewall Rules with iptables"
date: 2019-01-05 12:30:45 +01
categories: linux security
tags: iptables firewall security
---

## Intro

`iptables` is a powerful Linux firewall tool that provides fine-grained control over network traffic. Advanced configurations allow you to secure your system against sophisticated attacks, optimize traffic flow, and manage complex rule sets. This guide explores **advanced concepts** in `iptables`, including custom chains, rate limiting, NAT, and connection tracking to help you implement robust firewall rules.

---

## Step 1: Setting Up Basic Policies

Before diving into advanced rules, configure default policies to secure your system.

### **1.1 Default Drop Policy**

Block all incoming and forwarding traffic by default:

```sh
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

### **1.2 Allow Loopback Traffic**

Permit traffic on the loopback interface:

```sh
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
```

### **1.3 Drop Invalid Packets**

Block malformed or invalid packets:

```sh
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
```

---

## Step 2: Custom Chains for Better Organization

Custom chains help organize rules for easier management.

### **2.1 Create Custom Chains**

Create `whitelist` and `blacklist` chains for trusted and untrusted IPs:

```sh
iptables -N whitelist
iptables -N blacklist
```

### **2.2 Add Rules to Custom Chains**

Add IPs to the respective chains:

```sh
# Whitelist trusted IPs
iptables -A whitelist -s 192.168.1.100 -j ACCEPT

# Blacklist malicious IPs
iptables -A blacklist -s 203.0.113.51 -j DROP
```

### **2.3 Reference Custom Chains**

Direct traffic through the custom chains:

```sh
iptables -A INPUT -j whitelist
iptables -A INPUT -j blacklist
```

---

## Step 3: Rate Limiting for Attack Mitigation

Rate limiting helps prevent brute-force attacks and DoS attempts.

### **3.1 Limit SSH Login Attempts**

Restrict SSH login attempts to 5 per minute:

```sh
iptables -A INPUT -p tcp --dport 22 \
-m conntrack --ctstate NEW \
-m recent --set

iptables -A INPUT -p tcp --dport 22 \
-m conntrack --ctstate NEW \
-m recent --update --seconds 60 --hitcount 5 -j DROP
```

### **3.2 Limit ICMP (Ping) Traffic**

Prevent ping flood attacks by limiting ICMP requests:

```sh
iptables -A INPUT -p icmp \
-m limit --limit 1/s --limit-burst 5 \
-j ACCEPT

iptables -A INPUT -p icmp -j DROP
```

---

## Step 4: Stateful Packet Filtering with Connection Tracking

Stateful inspection allows `iptables` to track connection states.

### **4.1 Allow Established Connections**

Permit packets from established or related connections:

```sh
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### **4.2 Block Non-SYN Packets for New Connections**

Prevent non-SYN packets from initiating new connections:

```sh
iptables -A INPUT -p tcp ! --syn \
-m conntrack --ctstate NEW \
-j DROP
```

---

## Step 5: Protect Against Common Attacks

### **5.1 Drop NULL Packets**

Block NULL packets often used in reconnaissance scans:

```sh
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
```

### **5.2 Block XMAS Packets**

Drop XMAS packets used in stealth scans:

```sh
iptables -A INPUT \
-p tcp --tcp-flags ALL FIN,PSH,URG \
-j DROP
```

### **5.3 Mitigate SYN Floods**

Limit the rate of incoming SYN packets:

```sh
iptables -A INPUT \
-p tcp --syn \
-m limit --limit 20/s --limit-burst 50 \
-j ACCEPT

iptables -A INPUT \
-p tcp --syn \
-j DROP
```

---

## Step 6: Network Address Translation (NAT)

NAT is essential for routing traffic between private and public networks.

### **6.1 Enable Masquerading for Outbound Traffic**

Allow devices on a private network to access the internet:

```sh
iptables -t nat -A POSTROUTING \
-o eth0 \
-j MASQUERADE
```

### **6.2 Port Forwarding**

Forward traffic from port 8080 on the host to port 80 on an internal server:

```sh
iptables -t nat -A PREROUTING \
-p tcp --dport 8080 \
-j DNAT --to-destination 192.168.1.10:80

iptables -A FORWARD \
-p tcp --dport 80 \
-d 192.168.1.10 \
-j ACCEPT
```

---

## Step 7: Logging and Monitoring

Logging helps identify suspicious activity and debug firewall rules.

### **7.1 Log Dropped Packets**

Log dropped packets for analysis:

```sh
iptables -A INPUT \
-m limit --limit 5/min \
-j LOG --log-prefix "Dropped Packet: " --log-level 4

iptables -A INPUT \
-j DROP
```

### **7.2 Monitor Active Rules**

List all active `iptables` rules:

```sh
iptables-save | less
```

---

## Step 8: Save and Restore Rules

Ensure your rules persist across reboots.

### **8.1 Save Rules**

Save the current configuration:

```sh
sudo iptables-save > /etc/iptables/rules.v4
```

### **8.2 Restore Rules on Boot**

Restore rules automatically during startup by adding the following command to your system's boot scripts or using a service like `netfilter-persistent`:

```sh
sudo iptables-restore < /etc/iptables/rules.v4
```

---

## Conclusion

Advanced `iptables` configurations provide robust network security by enabling features like custom chains, rate limiting, stateful packet filtering, and NAT configurations. By implementing these techniques with concrete examples, you can protect your Linux servers against sophisticated threats while maintaining flexibility in managing network traffic.
