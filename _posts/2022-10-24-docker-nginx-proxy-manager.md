---
layout: post
title: "Docker - Install Nginx Proxy Manager"
date: 2022-10-24 22:14:09 +01
categories: docker
tags: nginx hosting
---

## Intro

### Install docker

We will be running the Nginx Proxy Manager in docker stack. Hence, we will be needing a working docker installation. You can follow another one of my posts on [Docker - Install and Configure](https://harryvasanth.github.io/posts/docker-install-configure/).

### Set up Nginx Proxy Manager

Create a new file `docker-compose.yml` file, please refer to the [Nginx Proxy Manager documentation](https://nginxproxymanager.com/guide/).

Example Docker-Compose File:

```yml
version: "3"
services:
  app:
    image: "jc21/nginx-proxy-manager:latest"
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    environment:
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm"
      DB_MYSQL_NAME: "npm"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
  db:
    image: "jc21/mariadb-aria:latest"
    environment:
      MYSQL_ROOT_PASSWORD: "npm"
      MYSQL_DATABASE: "npm"
      MYSQL_USER: "npm"
      MYSQL_PASSWORD: "npm"
    volumes:
      - ./data/mysql:/var/lib/mysql
```

#### Start the Nginx Proxy Manager

```bash
docker-compose up -d
```

#### Login to the web UI of NGINX proxy manager

Now we can log in to the web UI. Simply use your browser to connect to your server by using the IP address or an FQDN and connect on port `81`. Log in with the username `admin@example`.com and the password `changeme`. Next, you should change your username and password, and that’s it!
