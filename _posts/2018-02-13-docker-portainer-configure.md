---
layout: post
title: "Docker - Portainer Configuration"
date: 2018-02-13 08:31:04 +01
categories: docker
tags: portainer hosting
---

## Introduction

Portainer is a powerful management tool for Docker, providing an intuitive web interface to manage your containers, images, volumes, and networks. In this guide, we will explore advanced configurations and optimizations for a more robust Portainer setup.

### Prerequisites

Ensure that Docker is installed on your system. If not, you can refer to my post on [Docker - Install and Configure](https://harryvasanth.com/posts/docker-install-config/).

## Installation

Let's start by creating a Docker Compose file for Portainer. Create a file named `docker-compose.yml` and add the following content:

```yaml
version: "3"
services:
  portainer:
    image: portainer/portainer-ce
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    restart: always

volumes:
  portainer_data:
```

Save the file and run Portainer with the following command:

```bash
docker-compose up -d
```

## Access Portainer

Open your browser and navigate to `http://your-server-ip:9000`. Set up the initial admin user and password.

## Advanced Configurations

### Use HTTPS with a Reverse Proxy

To secure your Portainer instance, it's recommended to use HTTPS. Set up a reverse proxy like Nginx or Traefik, and obtain an SSL certificate.

Example Nginx configuration:

```nginx
server {
    listen 80;
    server_name portainer.yourdomain.com;

    location / {
        proxy_pass http://your-server-ip:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Secure Portainer with Authentication

Add an authentication layer to Portainer using a reverse proxy or Portainer's built-in authentication. Adjust the Portainer service in the `docker-compose.yml` file:

```yaml
version: "3"
services:
  portainer:
    image: portainer/portainer-ce
    ports:
      - "9000:9000"
    environment:
      - "SERVER_AGENT_SECRET=your-secret-key"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    restart: always
```

Replace `your-secret-key` with a strong secret.

### Backup Portainer Data

Set up regular backups for Portainer data. Create a backup script, and schedule it using cron.

Example backup script (`backup.sh`):

```bash
#!/bin/bash

docker run --rm -v portainer_data:/source -v /path/to/backup:/backup alpine tar -czf /backup/portainer_backup_$(date +"%Y%m%d%H%M%S").tar.gz -C /source .
```

Schedule the script to run daily:

```bash
0 3 * * * /path/to/backup.sh
```

## Conclusion

By following these advanced configurations, you can enhance the security and reliability of your Portainer setup. Explore more options in the official Portainer documentation to tailor the setup according to your specific requirements.

Feel free to reach out if you have any questions or need further assistance!
