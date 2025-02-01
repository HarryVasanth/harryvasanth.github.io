---
layout: post
title: "Docker - Install and Configure"
date: 2017-06-21 12:01:20 +01
categories: docker
tags: hosting
---

## Installation

The Docker installation package available in the official Ubuntu repository may not be the latest version. To ensure we get the latest version, we’ll install Docker from the official Docker repository. To do that, we’ll add a new package source, add the `GPG key` from Docker to ensure the downloads are valid, and then install the package.

First, install docker prerequisites:

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg
```

Get and run the docker setup script to install docker:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
```

## Configuration

If you want to avoid typing `sudo` whenever you run the `docker` command, add your username to the docker group:

```bash
sudo usermod -aG docker <user.name>
```
