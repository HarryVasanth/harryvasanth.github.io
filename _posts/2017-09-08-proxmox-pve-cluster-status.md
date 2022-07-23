---
layout: post
title: "Proxmox - Cluster Status"
date: 2016-01-21 15:24:03 +01
categories: proxmox
tags:  proxmox-ve
---

## Check Proxmox VE cluster status

This command will let you know the current status of Proxmox VE Cluster:

```bash
root@pve:~# pvecm status
Quorum information
------------------
Date:             Fri Jun 23 14:54:18 2017
Quorum provider:  corosync_votequorum
Nodes:            2
Node ID:          0x00000002
Ring ID:          1/32
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   2
Highest expected: 2
Total votes:      2
Quorum:           2  
Flags:            Quorate 

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 192.168.10.26
0x00000002          1 192.168.10.68 (local)
```

## Fix unresponsive Proxmox VE cluster

If a Proxmox node in the PVE cluster shows the unknown status (greyed out with `?`) as well as all the VMs of that node. We can restart the following services of that particular node through CLI to reload the node and its GUI elements.

```bash
sudo service pvestatd restart && sudo service pvedaemon restart && sudo service pveproxy restart
```
