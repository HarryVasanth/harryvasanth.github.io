---
layout: post
title: "Proxmox - Remove a Node from PVE Cluster"
date: 2016-07-11 17:38:43 +01
categories: proxmox
tags: proxmox-ve
---

## Introduction

These are the steps to remove a node from an existing Proxmox cluster.

>**Warning:** This tutorial may only be used if you want to delete permanently a node from an existing Proxmox cluster!
{: .prompt-danger }

## Migrate all virtual machines

Migrate all Virtual Machines to another active node.

>You can use the live migration feature if you have shared storage or offline migration if you only have local storage.
{: .prompt-tip }

## Check all active nodes

Display all active nodes in order to identify the name of the node you want to remove.

```bash
root@proxmox-node1:~# pvecm nodes
Membership information
----------------------
Nodeid Votes Name
1 1 proxmox-node0 (local)
2 1 proxmox-node1
3 1 proxmox-node2
4 1 proxmox-node3
```

## Shutdown node

Shutdown (permanently) the node that you want to remove.

> **Please be careful:** This is a **permanent** removal
>
>- Never restart the removed node
>- Don’t assign the local IP address of the removed node to a new node
>- Never assign the name of the removed node to a new node
{: .prompt-danger }

## Remove node internally

To remove the node from the proxmox cluster, connect to an active node, for example: `proxmox-node1`

```bash
root@proxmox:~# pvecm delnode NodeNameBashCopy
```

Example:

```bash
root@proxmox-node1:~# pvecm delnode proxmox-node2
```

## Remove node from the GUI

Remove the removed node from the proxmox GUI
Log in to an active node, for example: `proxmox-node1`

```bash
root@proxmox-node1:~# ls -l /etc/pve/nodes/
proxmox-node0 proxmox-node1 proxmox-node2 proxmox-node3
```

All nodes have is own directory (VM’s inventory, for example), the directory `/etc/pve/nodes/` is synced between all cluster nodes.The removed node is still visible in GUI until the node directory exists in the directory `/etc/pve/nodes/`. If you want to remove from Proxmox GUI the node previously deleted, you just need to delete the directory `/etc/pve/nodes/NodeName`.

```bash
root@proxmox-node1:~# mv /etc/pve/nodes/NodeName /root/NodeName
```

> **Warning:** Don’t do this unless you understand each step of the process and don’t mind putting the node in a state where you won’t easily be able to re-add it back to a cluster. Also note that after doing this, the containers on the node do not start up automatically, even though their config file says they are supposed to.
{: .prompt-danger }
