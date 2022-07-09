---
layout: post
title: "Linux - Mount An NFS Target"
date: 2017-10-29 18:55:22 +00
categories: linux
tags:  general
---

## Installation

First of all, we need to install the `nfs-common` dependencies:

```bash
sudo apt install nfs-common -y
```

Then we can list all available NFS targets from a remote server:

```bash
sshowmount -e 192.168.1.10
/mnt/nfs_pool_00/documents           192.168.1.10
/mnt/nfs_pool_01/backups/images      192.168.1.10,10.10.1.10
```

## Configuration

Once we determine the appropriate mount, we can go ahead and create a local mounting point and mount the remote NFS location:

```bash
sudo mkdir /mnt/documents
sudo mount 192.168.1.10:/mnt/nfs_pool_00/documents /mnt/documents
```

## Verification

Finally, we can verify if it’s mounted properly and writable:

```bash
sudo df -h
sudo mkdir /mnt/documents/misc
sudo touch /mnt/documents/misc/test.txt
sudo ls -la /mnt/documents/
```
