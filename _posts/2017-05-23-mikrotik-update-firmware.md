---
layout: post
title: "Mikrotik - Update Firmware"
date: 2017-05-23 11:12:03 +00
categories: mikrotik
tags:  router-os
---

First of all setup eamil alerts to be sent regarding the new available updates:

```c
/tool e-mail set address=mail.harryvasanth.com from=harry@harryvasanth.com
/system script add dont-require-permissions=no name=UpdateAlert owner=admin policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon source="/system package update\r\
   \ncheck-for-updates once\r\
   \n\r\
   \n:delay 3s;\r\
   \n\r\
   \n:if ( [get status] = \"New version is available\") do={ \r\
   \n\r\
   \n:local newVer [get latest-version]\r\
   \n/tool e-mail send to=\"harry@harryvasanth.com\" subject=\"New RouterOS firmware available for router \$[/system identity get name]\" body=\"Upgrading RouterOS on router \$[/system identity get name] from \$[/system package update get installed-version] to \$[/system package update get latest-version] (channel:\$[/system package update get channel])\"\r\
   \n  \r\
   \n}\r\
   \n"
```
{: file='Mikrotik Terminal'}

Reference: [Link](https://forum.mikrotik.com/viewtopic.php?t=111005)

Now lets setup automated update:

```c
 /system scheduler add name=UpgradeFirmware on-event="if ([/system routerboard get current-firmware] != [/system routerboard get upgrade-firmware]) do={\r\
   \n/system routerboard upgrade\r\
   \n:delay 1\r\
   \n/system reboot\r\
   \n}\r\
   \n" policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive start-time=startup
```
{: file='Mikrotik Terminal'}

Reference: [Link](https://wiki.arditi.pt/index.php/Setup_automated_routerboard_firmware_upgrade)

Additionally, enable SNMP to be monitored by network monitoring software:

```c
/snmp set enabled=yes
```
{: file='Mikrotik Terminal'}