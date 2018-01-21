---
layout: post
title: "Docker - Traefik Configuration"
date: 2018-01-21 21:48:12 +01
categories: docker
tags: traefik hosting
---

## Intro

### Install Docker

Ensure you have a working Docker installation on your server. If not, follow my guide on [Docker - Install and Configure](https://harryvasanth.com/posts/docker-install-config/).

### Set up Traefik

Create a new file named `docker-compose.yml` for Traefik. Refer to the [official Traefik documentation](https://doc.traefik.io/traefik/getting-started/configuration-overview/) for additional configuration options.

Example Docker-Compose File:

```yml
version: "3"
services:
  traefik:
    image: "traefik:v2.5"
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
```

#### Start Traefik

```bash
docker-compose up -d
```

#### Access Traefik Dashboard

You can now access the Traefik dashboard by navigating to `http://your-server-ip:8080` in your browser. Note that the `--api.insecure=true` flag is used for simplicity; in a production environment, consider securing the API.

## Configuration

Now, let's set up a sample service using Traefik. Create a new file named `whoami.yml`:

```yml
version: "3.3"
services:
  whoami:
    image: "containous/whoami"
    labels:
      - "traefik.http.routers.whoami.rule=Host(`whoami.yourdomain.com`)"
```

Replace `yourdomain.com` with your actual domain.

#### Deploy the Whoami Service

```bash
docker-compose -f whoami.yml up -d
```

Visit `http://whoami.yourdomain.com` to see the Traefik-managed service.

## Conclusion

Congratulations! You've successfully set up Traefik as a reverse proxy using Docker. Explore the [Traefik documentation](https://doc.traefik.io/traefik/) for advanced configuration options and security measures.
