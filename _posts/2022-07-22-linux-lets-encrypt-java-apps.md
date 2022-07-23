---
layout: post
title: "Linux - Let's Encrypt for Java Apps"
date: 2022-07-22 19:12:22 +01
categories: linux
tags:  hosting
---
## Intro

Install certbot as mentioned in my previous post on [Linux - Install Certbot](https://harryvasanth.github.io/posts/linux-install-certbot/).

By default, `certbot` retrieves a certificate and installs it immediately on the web server by adding an extra parameter, eg. `--apache` for Apache HTTP Server. For the case of [Docker - Demo CAS Server,](https://harryvasanth.github.io/posts/docker-demo-cas-server/) it is just enough to retrieve only a certificate. This is done by adding the `certonly` parameter to the command as follows:
