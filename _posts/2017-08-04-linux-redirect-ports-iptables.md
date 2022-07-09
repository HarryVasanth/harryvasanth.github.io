---
layout: post
title: "Linux - Redirect Ports Using IPTables"
date: 2017-08-04 23:28:56 +00
categories: linux
tags:  general
---

In order to redirect ports, we need to add our rules in `PREROUTING` chain. So run below command:

```bash
[root@harryvasanth.com ~]# iptables -t nat -A PREROUTING -i ens18 -p tcp --dport 80 -j REDIRECT --to-port 8080
```

If we have an interface name other than `ens18` then we need to edit we command accordingly. We can even add we source and destinations as well in same command using `--src` and `--dst` options. Without them, it's assumed to any source and any destination.

## Check port redirection in iptable

Verify port redirect rule in iptables using below command:

```bash
[root@harryvasanth.com ~]# iptables -t nat -L -n -v
Chain PREROUTING (policy ACCEPT 1 packets, 30 bytes)
pkts bytes target     prot opt in     out     source               destination
84  5000 REDIRECT   tcp  --  ens18  *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 redir ports 8080
```

We can see port 80 is being redirected to port 8080 on the server. Note here target is REDIRECT. Do not get confused with port redirection with port forwarding.

## Save IPTables rules

To save iptables rules and make them persistent over reboots use the following command:

```bash
[root@harryvasanth.com ~]# iptables-save
```