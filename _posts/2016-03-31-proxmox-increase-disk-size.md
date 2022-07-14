---
layout: post
title: "Proxmox - Increase VM Disk Size"
date: 2016-03-31 12:11:15 +00
categories: proxmox
tags:  proxmox-ve
---

## Steps

These are the steps to increase the disk size of the VMs in Proxmox

1. Add new disk in Proxmox VE UI
2. Partition the new disk
3. Create a new physical device (using `pvcreate`)
4. Extend the existing volume group
5. Extend the logical volume
6. Extend the file system (`ext4` in my case)

## Check current storage

First lets check how much space left on the VM:

```bash
root@jupyterhub:~# df -h
Filesystem                       Size  Used Avail Use% Mounted on
udev                             2.0G     0  2.0G   0% /dev
tmpfs                            396M   11M  385M   3% /run
/dev/mapper/ubuntu1604--vg-root   18G   15G  1.8G  90% /
tmpfs                            2.0G  4.0K  2.0G   1% /dev/shm
tmpfs                            5.0M     0  5.0M   0% /run/lock
tmpfs                            2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/ubuntu1604--vg-home   20G   44M   19G   1% /home/jupyter
/dev/mapper/ubuntu1604--vg-log   465M  112M  325M  26% /var/log
/dev/sda1                        472M  155M  293M  35% /boot
tmpfs                            396M     0  396M   0% /run/user/2160
```

Now lets see all the available disks on the system after adding a new disk through Promox VE User Interface:

```bash
root@testvm:~# fdisk -l
Disk /dev/sda: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x93152316

Device     Boot   Start      End  Sectors  Size Id Type
/dev/sda1  *       2048   999423   997376  487M 83 Linux
/dev/sda2       1001470 20969471 19968002  9.5G  5 Extended
/dev/sda5       1001472 20969471 19968000  9.5G 8e Linux LVM


Disk /dev/sdb: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdc: 32 GiB, 34359738368 bytes, 67108864 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 6B4A0899-D3D5-4771-856D-EDB254829C91

Device     Start      End  Sectors Size Type
/dev/sdc1   2048 67108830 67106783  32G Linux filesystem


Disk /dev/mapper/ubuntu1604--vg-root: 17.5 GiB, 18815647744 bytes, 36749312 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/ubuntu1604--vg-swap: 952 MiB, 998244352 bytes, 1949696 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/ubuntu1604--vg-log: 488 MiB, 511705088 bytes, 999424 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/ubuntu1604--vg-home: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

## Partitioning the new disk

As you can see the `64 GiB` disk is present without any partitioning. Lets go ahead and partition that disk:

```bash
root@testvm:~# fdisk /dev/sdb

Welcome to fdisk (util-linux 2.27.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x193c65a1.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-134217727, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-134217727, default 134217727):

Created a new partition 1 of type 'Linux' and of size 64 GiB.

Command (m for help): p
Disk /dev/sdb: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x193c65a1

Device     Boot Start       End   Sectors Size Id Type
/dev/sdb1        2048 134217727 134215680  64G 83 Linux

Command (m for help): p
Disk /dev/sdb: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x193c65a1

Device     Boot Start       End   Sectors Size Id Type
/dev/sdb1        2048 134217727 134215680  64G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

Now let’s create a new physical device with the newly partitioned disk using `pvcreate` and view it using `pvdisplay`:

```bash
root@testvm:~# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created

root@jupyterhub:~# pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda5
  VG Name               ubuntu1604-vg
  PV Size               9.52 GiB / not usable 2.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              2437
  Free PE               0
  Allocated PE          2437
  PV UUID               qNCelA-nJDf-NX7Q-ZkK4-iR4M-vQho-mfFqA1

  --- Physical volume ---
  PV Name               /dev/sdc1
  VG Name               ubuntu1604-vg
  PV Size               32.00 GiB / not usable 2.98 MiB
  Allocatable           yes
  PE Size               4.00 MiB
  Total PE              8191
  Free PE               662
  Allocated PE          7529
  PV UUID               fu3bxw-oCCH-jOh1-428b-lAQf-DCBe-E6KQiv

  "/dev/sdb1" is a new physical volume of "64.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb1
  VG Name
  PV Size               64.00 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               F0FsRP-ROTm-xbMv-7I3D-E0Jh-mfBw-rM4mn4
```

## Check current volume group

Thereafter, let’s go ahead and check the current volume groups using `vgdisplay`:

```bash
root@testvm:~# vgdisplay
  --- Volume group ---
  VG Name               ubuntu1604-vg
  System ID
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  12
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                4
  Open LV               4
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               41.52 GiB
  PE Size               4.00 MiB
  Total PE              10628
  Alloc PE / Size       9966 / 38.93 GiB
  Free  PE / Size       662 / 2.59 GiB
  VG UUID               BR84gF-JBKN-ndcp-At3G-l9UB-4CAs-KVyyta
```

## Extend current volume group

Now we can extend it using `vgextend` and verify it using `vgdisplay`:

```bash
root@testvm:~# vgextend ubuntu1604-vg /dev/sdb1
  Volume group "ubuntu1604-vg" successfully extended

root@testvm:~# vgdisplay
  --- Volume group ---
  VG Name               ubuntu1604-vg
  System ID
  Format                lvm2
  Metadata Areas        3
  Metadata Sequence No  13
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                4
  Open LV               4
  Max PV                0
  Cur PV                3
  Act PV                3
  VG Size               105.51 GiB
  PE Size               4.00 MiB
  Total PE              27011
  Alloc PE / Size       9966 / 38.93 GiB
  Free  PE / Size       17045 / 66.58 GiB
  VG UUID               BR84gF-JBKN-ndcp-At3G-l9UB-4CAs-KVyyta
```

Thereafter, lets go ahead and check the logical volumes using `lvdisplay`:

```bash
root@testvm:~# lvdisplay
  --- Logical volume ---
  LV Path                /dev/ubuntu1604-vg/swap
  LV Name                swap
  VG Name                ubuntu1604-vg
  LV UUID                0SYO9N-bAdU-vfGR-soF1-vvcp-Bg8v-EPrJEL
  LV Write Access        read/write
  LV Creation host, time ubuntu1604, 2017-06-27 15:12:58 +0100
  LV Status              available
  # open                 2
  LV Size                952.00 MiB
  Current LE             238
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:1

  --- Logical volume ---
  LV Path                /dev/ubuntu1604-vg/log
  LV Name                log
  VG Name                ubuntu1604-vg
  LV UUID                fXp0kR-TYBp-mdiS-sYXd-ja2S-dABf-ThjEf0
  LV Write Access        read/write
  LV Creation host, time ubuntu1604, 2017-06-27 15:13:14 +0100
  LV Status              available
  # open                 1
  LV Size                488.00 MiB
  Current LE             122
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:2

  --- Logical volume ---
  LV Path                /dev/ubuntu1604-vg/root
  LV Name                root
  VG Name                ubuntu1604-vg
  LV UUID                WwteZj-vPhz-3iNC-26Dg-Ju36-o6xe-75WRto
  LV Write Access        read/write
  LV Creation host, time ubuntu1604, 2017-06-27 15:13:50 +0100
  LV Status              available
  # open                 1
  LV Size                17.52 GiB
  Current LE             4486
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:0

  --- Logical volume ---
  LV Path                /dev/ubuntu1604-vg/home
  LV Name                home
  VG Name                ubuntu1604-vg
  LV UUID                OQHBcC-aEeD-ePLq-ezMU-3OgV-JPyn-2yarKe
  LV Write Access        read/write
  LV Creation host, time ubuntu1604, 2017-09-27 12:16:53 +0100
  LV Status              available
  # open                 1
  LV Size                20.00 GiB
  Current LE             5120
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:3
```

## Extend the root volume

Lets extend the root volume by `10 GiB` using `lvextend`:

```bash
root@testvm:~# lvextend --resizefs -L+10G /dev/ubuntu1604-vg/root
  Size of logical volume ubuntu1604-vg/root changed from 22.52 GiB (5766 extents) to 32.52 GiB (8326 extents).
  Logical volume root successfully resized.
resize2fs 1.42.13 (17-May-2015)
Filesystem at /dev/mapper/ubuntu1604--vg-root is mounted on /; on-line resizing required
old_desc_blocks = 2, new_desc_blocks = 3
The filesystem on /dev/mapper/ubuntu1604--vg-root is now 8525824 (4k) blocks long.
```

## Allocate entire disk space to the volume

In case we want the entire disk space to extended you can simply assign 100% of the free space.

```bash
root@testvm:~# lvextend --resizefs -l +100%FREE /dev/ubuntu1604-vg/root
```
