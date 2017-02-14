---
layout: post
title: "Linux - Install Fail2ban"
date: 2017-02-14 17:02:14 +01
categories: linux hosting
tags: hosting
---

## Installation

First, install Fail2ban using `apt`:

```bash
sudo apt update
sudo apt install fail2ban
```

## Configuration

The default Fail2ban installation comes with two configuration files, `/etc/fail2ban/jail.conf` and `/etc/fail2ban/jail.d/defaults-debian.conf`. It is not recommended to modify these files as they may be overwritten when the package is updated.

Fail2ban reads the configuration files in the following order. Each `.local` file overrides the settings from the `.conf` file:

```bash
/etc/fail2ban/jail.conf
/etc/fail2ban/jail.d/*.conf
/etc/fail2ban/jail.local
/etc/fail2ban/jail.d/*.local
```

For most users, the easiest way to configure Fail2ban is to copy the `jail.conf` to jail.local and modify the `.local` file. More advanced users can build a `.local` configuration file from scratch. The `.local` file doesn’t have to include all settings from the corresponding `.conf` file, only those you want to override.

Create a `.local` configuration file from the default `jail.conf` file:

```bash
sudo cp /etc/fail2ban/jail.{conf,local}
```

Then go ahead and edit the `jail.local` file with the following values:

```conf
# "bantime" is the number of (s)econds/(m)inutes/(h)ours/(d)ays that a host is banned.
bantime  = 7d
# A host is banned if it has generated "maxretry" during the last "findtime"
# (s)econds/(m)inutes/(h)ours/(d)ays.
findtime  = 1h
# "maxretry" is the number of failures before a host gets banned within the "findtime".
maxretry = 6
```

## Verification

Then go ahead and restart the Fail2ban service and see if the status is ok.

```bash
sudo service fail2ban restart && sudo service fail2ban status
```
