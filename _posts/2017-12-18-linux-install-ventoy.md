---
layout: post
title: "Linux - Install Ventoy"
date: 2017-12-18 22:57:11 +01
categories: linux ventoy
tags: ventoy
---

## Introduction

Ventoy is an open-source tool that allows you to create a multiboot USB drive, enabling you to boot multiple ISO files directly from the USB without the need to repeatedly recreate the USB drive. This tutorial will guide you through the installation process on both Windows and Linux.

### Windows Installation

#### Step 1: Download Ventoy

Visit the official Ventoy GitHub releases page and download the latest Windows version: [Ventoy Releases](https://github.com/ventoy/Ventoy/releases)

#### Step 2: Install Ventoy on USB Drive

1. Insert your USB drive into your Windows computer.
2. Extract the downloaded Ventoy ZIP file.
3. Run the Ventoy2Disk.exe application.
4. Select your USB drive from the dropdown menu.
5. Click the "Install" button to install Ventoy on the USB drive.

#### Step 3: Add ISO Files

1. After installation, copy your desired ISO files to the USB drive.
2. Ventoy will detect and list the ISO files on the main screen.

### Linux Installation

#### Step 1: Download Ventoy

Open a terminal and run the following commands to download and extract Ventoy:

```bash
wget https://github.com/ventoy/Ventoy/releases/latest/download/ventoy-x86_64.tar.gz
tar xzvf ventoy-x86_64.tar.gz
```

#### Step 2: Install Ventoy on USB Drive

1. Insert your USB drive into your Linux computer.
2. Navigate to the Ventoy directory.
3. Run the following command to install Ventoy on the USB drive:

```bash
sudo sh Ventoy2Disk.sh -I /dev/sdX
```

Replace `/dev/sdX` with your actual USB drive identifier.

#### Step 3: Add ISO Files

1. Copy your desired ISO files to the USB drive.
2. Ventoy will detect and list the ISO files on the main screen.

## Usage

1. Plug the Ventoy USB drive into your target system.
2. Boot from the USB drive.
3. Select the desired ISO from the Ventoy boot menu.

Ventoy simplifies the process of creating a versatile multiboot USB drive, making it a convenient tool for system administrators and enthusiasts alike.
