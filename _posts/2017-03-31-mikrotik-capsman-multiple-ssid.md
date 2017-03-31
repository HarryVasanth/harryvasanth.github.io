---
layout: post
title: "Mikrotik - Mutiple SSID with CAPsMAN"
date: 2017-03-31 09:49:22 +01
categories: mikrotik
tags:  router-os
---

## Pre-requisites
The ports on the switches where the APs are to be connected, should be configured with the following VLANs:

```yml
VLAN 6 (HOME): not tagged
VLAN 12 (HOME-Guest): tagged
```
{: file='RouterOS'}

## Quickset Mode
In this mode configure the following settings as follows:

**Configuration**

```yml
Mode: Bridge
```
{: file='RouterOS'}

**System**

```yml
Password: (enter new admin password)
```
{: file='RouterOS'}

Press the Apply Configuration button.

## Terminal

**CAPs-Man**

- Create appropriate CAPsMan datapath template

    ```c
    /caps-man datapath add client-to-client-forwarding=yes local-forwarding=yes name=datapath-HOME vlan-id=6 vlan-mode=no-tag
    /caps-man datapath add client-to-client-forwarding=yes local-forwarding=yes name=datapath-Guest vlan-id=12 vlan-mode=use-tag
    ```
    {: file='Mikrotik Terminal'}

- Create appropriate CAPsMan security template
  
    ```c
    /caps-man security add authentication-types=wpa-psk,wpa2-psk name=security-HOME passphrase=h0m3w1f1
    /caps-man security add authentication-types=wpa-psk,wpa2-psk name=security-Guest passphrase=home-guest
    ```
    {: file='Mikrotik Terminal'}

- Create appropriate CAPsMan configurations
  
    ```c
    /caps-man configuration add country=portugal datapath=datapath-HOME name=config-HOME security=security-HOME ssid=HOME 
    /caps-man configuration add country=portugal datapath=datapath-Guest name=config-Guest security=security-Guest ssid=HOME-Guest
    ```
    {: file='Mikrotik Terminal'}

    We are going to create a single CAPsMAN provisioning rule to create the HOME and the HOME-Guest SSIDs on a single device, each connects CAP is going to create these SSIDs automatically.

    ```c
    /caps-man provisioning add action=create-dynamic-enabled master-configuration=config-HOME slave-configurations=config-Guest
    ```
    {: file='Mikrotik Terminal'}

- Enable the CAPsMAN manager

    ```c
    /caps-man manager set enabled=yes package-path=/ upgrade-policy=suggest-same-version
    ```
    {: file='Mikrotik Terminal'}

- Enable the Wireless of the AP to be managed by the CAP

    ```c
    /interface wireless cap
    ```
    {: file='Mikrotik Terminal'}