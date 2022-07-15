---
layout: post
title: "Mikrotik - Configure cAP"
date: 2017-03-01 05:11:29 +00
categories: mikrotik
tags:  router-os
---

## Requirements

Requirements are to have two WiFi networks, one with **SSID** `HOME` and connected to the Home `VLAN (6)`, and a guest WiFI network, with **SSID** `HOME-Guest` and connected to the Guest `VLAN (12)`.

This example is based on Mikrotik cAP ac

## Initial Configuration

Quickset is a simple configuration wizard page that prepares your router in a few clicks. It is the first screen a user sees when opening the router in a web browser.

```yml
Select Quickset Mode: WISP AP
```
{: file='RouterOS'}

In this mode configure the following settings as follows:

## Configuration

```yml
Mode: Bridge
```
{: file='RouterOS'}

## Wireless

```yml
Wireless Protocol: (leave) 802.11
Network Name: HOME
Frequency: (leave) auto
Band: (leave) 5GHz-A/N/AC
Cannel Width: (leave) 20/40/80MHz Ceee
Country: portugal
Security: WPA2
Encryption: aes ccm
WiFi Password: (enter wifi password)
```
{: file='RouterOS'}

## Bridge

```yml
Address Acquisition: static
IP address: 192.168.6.228
Netmask: 255.255.255.0
Gateway: 192.168.6.254
DNS Servers: 192.168.6.230
```
{: file='RouterOS'}

## System

```yml
Password: (enter new admin password)
```
{: file='RouterOS'}

Press the Apply Configuration button. We should have a working WiFi AP on the HOME network. However, the configuration is no complete. The configuration continues using WebFig and Terminal.

Add necessary VLAN interfaces on ethernet interfaces to make them as VLAN trunk or access ports:

```c
add interface=ether1 name=eth2-vlan6 vlan-id=6
add interface=ether1 name=eth1-vlan12 vlan-id=12
add interface=ether2 name=eth2-vlan12 vlan-id=12
```
{: file='Mikrotik Terminal'}

Add bridges for each VLAN

```c
add name=bridge-vlan6
add name=bridge-vlan12
```
{: file='Mikrotik Terminal'}

The Quickset configuration had set the IP address in ether2 interface, but since we have added bridges for each VLAN it is better (safer) to set the IP on the bridge bridge-vlan6.

In WebFig add Security Profile for the HOME-Guest Wi-Fi network:

Goto Wireless -> Security Profiles and press the Add New to create a new profile.

```yml
Name: home-guest
Mode: (leave) dynamic keys
Authentication Types: (leave) WPA PSK and WPA2 PSK
Unicast Ciphers: (leave) aes ccm
Group Ciphers: (leave) aes ccm
WPA Pre-Shared Key: h0m3w1f1
WPA2 Pre-Shared Key: h0m3w1f1
```
{: file='RouterOS'}

Press OK, to create new security profile.

In WebFig add virtual wireless interfaces for the HOME-Guest Wi-Fi:

Goto Wireless -> WiFi Interfaces and press the Add New and select Virtual to create a new interface.

```yml
Name: HOME-Guest-2G
SSID: HOME-Guest
Master Interface: HOME-2G
Security Profile: home-guest
```
{: file='RouterOS'}

Press OK, to create new interface.

Goto Wireless -> WiFi Interfaces and press the Add New and select Virtual to create a new interface.

```yml
Name: HOME-Guest-5G
SSID: HOME-Guest
Master Interface: HOME-5G
Security Profile: home-guest
```
{: file='RouterOS'}

Press OK, to create new interface.

Add VLAN interfaces to their corresponding bridges and ethernet interfaces where untagged traffic is necessary

```c
/interface bridge port
add bridge=bridge-vlan6 interface=ether1
add bridge=bridge-vlan12 interface=eth1-vlan12
add bridge=bridge-vlan12 interface=HOME-Guest-2G 
add bridge=bridge-vlan12 interface=HOME-Guest-5G
```
{: file='Mikrotik Terminal'}