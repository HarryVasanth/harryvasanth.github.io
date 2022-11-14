---
layout: post
title: "Proxmox - Resize PVE Root"
date: 2016-01-21 15:24:03 +01
categories: proxmox
tags:  proxmox-ve
---

## Why?

When installing proxmox from iso its give a lot storage to `pve-data` which is a `volume-storage` and not a lot for `pve-root`, and `pve-root` it’s where your filesystem is mounted. 

## Check current storage

First of all, lets check how much we have in all of our `volume-storage`:

```bash
lvdisplay
```

The output should be something like this:

```shell
root@pve:~# lvdisplay 
  --- Logical volume ---
  LV Path                /dev/pve/swap
  LV Name                swap
  VG Name                pve
  LV UUID                niOT4X-rcyP-dKCD-sqAK-SWxy-dto9-vlbNjj
  LV Write Access        read/write
  LV Creation host, time proxmox, 2020-05-04 10:39:46 +0300
  LV Status              available
  # open                 2
  LV Size                7.00 GiB
  Current LE             1792
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
   
  --- Logical volume ---
  LV Path                /dev/pve/root
  LV Name                root
  VG Name                pve
  LV UUID                OrPb0H-h1lm-OzZ8-C02y-6FHy-RAuV-ppfrQp
  LV Write Access        read/write
  LV Creation host, time proxmox, 2020-05-04 10:39:46 +0300
  LV Status              available
  # open                 1
  LV Size                14.75 GiB
  Current LE             3776
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:1
   
  --- Logical volume ---
  LV Name                data
  VG Name                pve
  LV UUID                3O0Z5M-uV0c-tCXH-8Sz0-XZrk-2rZY-qQkvuR
  LV Write Access        read/write
  LV Creation host, time proxmox, 2020-05-04 10:39:46 +0300
  LV Pool metadata       data_tmeta
  LV Pool data           data_tdata
  LV Status              available
  # open                 0
  LV Size                28.37 GiB
  Allocated pool data    0.00%
  Allocated metadata     0.02%
  Current LE             7263
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:4
```

As we can see there is `28.37 GiB` in `pve-data` and only `14.75 GiB` in `pve-root`.

## Resize

For example, if we want to resize our `pve-data` to be `10 GiB` and also give `pve-root` the rest of the space, we can do it by do it as follows:

```bash
# remove pve-data logical volume.
lvremove /dev/pve/data -y

# create it again with new size.
lvcreate -L 10G -n data pve -T

# give pve-root all the other size.
lvresize -l +100%FREE /dev/pve/root

# resize pve-root file system
resize2fs /dev/mapper/pve-root
```

## Verfiy

Now you can check the new resized values by using:

```bash
lvdisplay
```
