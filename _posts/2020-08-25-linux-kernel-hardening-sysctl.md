---
layout: post
title: "Linux - Security: Hardening the Kernel with sysctl and Kernel Self-Protection"
date: 2020-08-25 21:31:33 +01
categories: linux security
tags: linux kernel security hardening sysctl system-administration
---

## Beyond the Firewall: The Kernel Attack Surface

Most security discussions revolve around firewalls and application-level vulnerabilities. However, for a highly skilled Linux administrator, the most critical defensive layer is the kernel itself. The Linux kernel is a massive, complex piece of software with millions of lines of code, much of which was written for features you likely don't need in a production server environment. These unused features (like old networking protocols or debugging interfaces) often contain "Zero Day" vulnerabilities that can lead to local privilege escalation or container escapes.

In 2020, hardening the kernel using `sysctl` and the **Kernel Self-Protection Project (KSPP)** guidelines is a mandatory task for any production-facing infrastructure.

## 1. Hardening the Network Stack (TCP/IP)

The Linux network stack is highly configurable. We can use `sysctl` to mitigate common network-based attacks like SYN floods and source routing exploits.

```bash
# Ignore ICMP redirects (prevents MITM attacks)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0

# Ignore source-routed packets
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Enable SYN Cookies to protect against SYN Flood attacks
net.ipv4.tcp_syncookies = 1

# Log packets with impossible addresses (Martians)
net.ipv4.conf.all.log_martians = 1
```

## 2. Restricting Information Leakage (ASLR and dmesg)

Attackers often use system information to craft their exploits. By restricting access to kernel logs and memory addresses, we make their job significantly harder.

- **ASLR (Address Space Layout Randomization)**: Ensures that the memory addresses of system components are random for each execution.

```bash
kernel.randomize_va_space = 2
```

- **Restricting dmesg**: By default, any user can run `dmesg` to see kernel logs, which may contain sensitive hardware information or memory addresses.

```bash
kernel.dmesg_restrict = 1
```

## 3. Disabling 'Ptrace' for Unprivileged Users

`ptrace` is a powerful system call used for debugging (like `gdb` or `strace`). However, it can also be used by an attacker to inject code into a running process or steal sensitive information like API keys or memory content.

```bash
# Only allow root (or parent processes) to use ptrace
kernel.yama.ptrace_scope = 1
```

## 4. Hardening the Just-In-Time (JIT) Compiler

The eBPF JIT compiler is a frequent source of kernel vulnerabilities. While eBPF is powerful, we should restrict its use to root and harden the generated code.

```bash
# Enable BGP JIT hardening
net.core.bpf_jit_harden = 2
```

## 5. Implementing Changes via /etc/sysctl.d/

Never apply changes directly to `/etc/sysctl.conf`. Instead, create a dedicated security configuration file to ensure your customisations are easy to audit and persist through system updates.

```bash
# /etc/sysctl.d/99-hardening.conf
kernel.kptr_restrict = 2
kernel.perf_event_paranoid = 3
fs.protected_fifos = 2
fs.protected_regular = 2
```

Apply the changes immediately:

```bash
sudo sysctl --system
```

## Summary: Defense in Depth

Hardening the kernel is not a "silver bullet" that stops all attacks, but it significantly increases the cost and complexity for an attacker. It is a critical component of a "Defense in Depth" strategy. A experienced administrator knows that every unused feature disabled is one less potential exploit path. By aligning your systems with the KSPP guidelines, you ensure that your infrastructure remains a "hard target" in an increasingly hostile threat landscape. This level of rigour is what separates a standard Linux server from a hardened production node in 2020.
