---
layout: post
title: "Mikrotik - Automate Backups"
date: 2017-04-01 22:09:02 +01
categories: mikrotik
tags:  router-os
---

## Why?

It is useful to have both textual and binary backups since the textual backup (without sensitive data, such as passwords) can be read by a human to understand the router configuration, while the latter (binary) can be used to restore configuration (with sensitive data) with ease. However, the binary configuration is more prone to corruption.

## Schedule: Textual Backup

```c
/system scheduler add comment="Backup running config in textual format without sensitive data" interval=1w name=runningConfigDump on-event="export file=/flash/mtikconf-nosensitive terse" policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon start-date=apr/10/2000 start-time=12:00:00
```
{: file='Mikrotik Terminal'}

## Schedule: Binary Backup

```c
/system scheduler add comment="Backup config as binary" interval=1w name=binBackup on-event="system backup save name=/flash/mtikAP.backup" policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon start-date=apr/10/2000 start-time=12:00:00
```
{: file='Mikrotik Terminal'}

## Script: Email Backups

Script: Email the backup file with system resource and health report (can be scheduled to run periodically)

```c
#Local Strings
:local rPrint [:tostr [/system resource print as-value]];
:local hPrint [:tostr [/system health print as-value]];

#Merged Strings
:local strMerged ("Resource: \n".$rPrint."\n\nHealth: \n".$hPrint);
:local strSplit value="";

:if ([:find $strMerged ";" -1] > 0) do={
 :for i from=0 to=([:len $strMerged] -1) step=1 do={
  :local escapeChar value=[:pick $strMerged $i];
  :if ($escapeChar = ";") do={ :set escapeChar value="\n" };
  :set strSplit value=($strSplit.$escapeChar);
 }
}

/tool e-mail send to="harry@harryvasanth.com" subject="Router backup - $[/system identity get name] - $[/system clock get date]" body="$strSplit" file=mtik.backup
```
{: file='Mikrotik Terminal'}