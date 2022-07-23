---
layout: post
title: "Linux - Port Forward Using IPTables"
date: 2017-07-28 22:34:02 +01
categories: linux
tags:  general
---

First of all, we need to check if **port forwarding** is enabled or not on our server. For better understanding, we will be using `ens18` as a reference interface and all our command executions will be related to `ens18` in this post.

## Check if port forwarding is enabled in Linux

Either we can use `sysctl` to check if forwarding is enabled or not. Use below command to check:

```bash
[root@harryvasanth.com ~]# sysctl -a | grep -i ens18.forwarding
net.ipv4.conf.ens18.forwarding = 0
net.ipv6.conf.ens18.forwarding = 0
```

Since both values are zero, **port forwarding** is disabled for `ipv4` and `ipv6` on interface `ens18`.
We can also use the process filesystem to check if **port forwarding** is enabled or not.

```bash
[root@harryvasanth.com ~]# cat /proc/sys/net/ipv4/conf/ens18/forwarding
0

[root@harryvasanth.com ~]# cat /proc/sys/net/ipv6/conf/ens18/forwarding
0
```

Again here process `FS` with zero values confirms **port forwarding** is disabled on our system. Now we need to first enable **port forwarding** on our system then we will configure **port forwarding** rules in `iptables`.

## Enable port forwarding in Linux

As we checked above, using the same methods we can enable **port forwarding** in Linux. But its recommended using `sysctl` command rather than replacing `0` by `1` in `proc` files.

Enable **port forwarding** in Linux using `sysctl` command:

```bash
[root@harryvasanth.com ~]# sysctl net.ipv4.conf.ens18.forwarding=1
net.ipv4.conf.ens18.forwarding = 1

[root@harryvasanth.com ~]# sysctl net.ipv6.conf.ens18.forwarding=1
net.ipv6.conf.ens18.forwarding = 1
```

To make it persistent over reboots, add parameters in `/etc/sysctl.conf` :

```bash
[root@harryvasanth.com ~]# echo "net.ipv4.conf.ens18.forwarding = 1">>/etc/sysctl.conf

[root@harryvasanth.com ~]# echo "net.ipv6.conf.ens18.forwarding = 1">>/etc/sysctl.conf

[root@harryvasanth.com ~]# sysctl -p
net.ipv4.conf.ens18.forwarding = 1
net.ipv6.conf.ens18.forwarding = 1
```

Now, we have **port forwarding** enabled on our server, we can go ahead with configuring **port forwarding** rules using `iptables`.

## Forward ports in Linux

Here we will forward tcp port `80` to tcp port `8080` on `192.168.1.1`. Do not get confused **port forwarding** with **port redirection**.We need to insert an entry in `PREROUTING` chain of iptables with `DNAT` target. Command will be as follows:

```bash
iptables -A PREROUTING -t nat -i ens18 -p tcp --dport 80 -j DNAT --to 192.168.1.1:8080
iptables -A FORWARD -p tcp -d 192.168.1.1 --dport 8080 -j ACCEPT
```

Change interface, IP and ports as per our requirement. The first command tells us to redirect packets coming to port `80` to IP `192.168.1.1` on port `8080`. Now packet also needs to go through `FORWARD` chain so we are allowing in in the second command. Now rules have been applied. We need to verify them.

## Check port forwarding IPTables rules

Command to verify **port forwarding** rules is:

```bash
[root@harryvasanth.com ~]# iptables -t nat -L -n -v
Chain PREROUTING (policy ACCEPT 1 packets, 30 bytes)
pkts    bytes   target      prot    opt in      out source      destination
  84    5000    REDIRECT    tcp     --  ens18   *   0.0.0.0/0   0.0.0.0/0   tcp dpt:80 to:192.168.1.1:8080
```

Here `REDIRECT` target means it’s a redirection rule. Since we have configured the forwarding rule we see the target as `DNAT`.

## Save IPTables rules

To save iptables rules and make them persistent over reboots use the following command:

```bash
[root@harryvasanth.com ~]# iptables-save
```
