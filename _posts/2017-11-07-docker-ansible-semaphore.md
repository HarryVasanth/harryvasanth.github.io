---
layout: post
title: "Docker - Install Ansible-Semaphore"
date: 2017-11-07 18:12:24 +01
categories: docker
tags: ansible semaphore
---

## Intro

### Install docker

We will be running the Ansible-Semaphore in docker stack. Hence, we will be needing a working docker installation. You can follow another one of my posts on [Docker - Install and Configure](https://harryvasanth.github.io/posts/docker-install-configure/).

### Set up Ansible-Semaphore

Create a new file `docker-compose.yml` file, please refer to the [Ansible-Semaphore documentation](https://docs.ansible-semaphore.com/administration-guide/installation).

Example Docker-Compose File:

```yml
services:
  mysql:
    image: mysql:8.0
    hostname: mysql
    volumes:
      - semaphore-mysql:/var/lib/mysql
    environment:
      - MYSQL_RANDOM_ROOT_PASSWORD=yes
      - MYSQL_DATABASE=semaphore
      - MYSQL_USER=semaphore
      - MYSQL_PASSWORD=secret-password # change!
    restart: unless-stopped
  semaphore:
    container_name: ansiblesemaphore
    image: semaphoreui/semaphore:v2.8.90
    user: "${UID}:${GID}"
    ports:
      - 3000:3000
    environment:
      - SEMAPHORE_DB_USER=semaphore
      - SEMAPHORE_DB_PASS=secret-password # change!
      - SEMAPHORE_DB_HOST=mysql
      - SEMAPHORE_DB_PORT=3306
      - SEMAPHORE_DB_DIALECT=mysql
      - SEMAPHORE_DB=semaphore
      - SEMAPHORE_PLAYBOOK_PATH=/tmp/semaphore/
      - SEMAPHORE_ADMIN_PASSWORD=secret-admin-password # change!
      - SEMAPHORE_ADMIN_NAME=admin
      - SEMAPHORE_ADMIN_EMAIL=admin@localhost
      - SEMAPHORE_ADMIN=admin
      - SEMAPHORE_ACCESS_KEY_ENCRYPTION= # add to your access key encryption !
      - ANSIBLE_HOST_KEY_CHECKING=false # (optional) change to true if you want to enable host key checking
    volumes:
      - ./inventory/:/inventory:ro
      - ./authorized-keys/:/authorized-keys:ro
      - ./config/:/etc/semaphore:rw
    restart: unless-stopped
    depends_on:
      - mysql
volumes:
  semaphore-mysql:
    driver: local
```

Modify the file by changing the database password, adding a strong admin password, and generating a new access key encryption. In order to protect sensitive information such as SSH keys or passwords in the database, a secret access key is required.

You can generate a strong password with the following command:

```bash
head -c32 /dev/urandom | base64
```

#### Start the Ansible-Semaphore

```bash
docker-compose up -d
```

#### Login to the web UI of Ansible-Semaphore

After the Ansible Semaphore stack has been started, we can log in to the web UI using our configured admin username and password. Once logged in, we can create our new project, and that's it!
