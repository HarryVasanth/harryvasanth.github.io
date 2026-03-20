---
layout: post
title: "Docker - Compose: Using 'extends' and 'include' to DRY up Large Microservice Architectures"
date: 2022-09-17 05:58:00 +01
categories: docker automation
tags: docker-compose devops automation best-practices
---

## The Problem: The 2000-Line docker-compose.yml

As your infrastructure grows, your `docker-compose.yml` becomes a nightmare to maintain. You have 20 services, and they all share the same logging config, the same environment variables, and the same network settings. Changing a single API key requires 20 edits.

## The Optimal Solution: The DRY Principle (Don't Repeat Yourself)

Docker Compose (especially V2) provides two powerful mechanisms for modularising your configuration: `extends` and the newer `include` (introduced in Docker Compose 2.20).

### Method 1: Using 'extends' for Template Services

Define a "base" service in a hidden file (e.g., `common.yml`):

```yaml
# common.yml
services:
  app-base:
    image: my-company/base-app:latest
    networks:
      - app-net
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
```

Then, extend it in your main file:

```yaml
# docker-compose.yml
services:
  web-api:
    extends:
      file: common.yml
      service: app-base
    ports:
      - "8080:80"
    environment:
      - SERVICE_NAME=api
```

### Method 2: Using 'include' for Multi-Repo setups

If different teams manage different parts of the stack, use `include`.

```yaml
# main-compose.yml
include:
  - path: ./networks/compose.yml
  - path: ./database-tier/compose.yml
  - path: ./frontend-tier/compose.yml

services:
  gateway:
    image: nginx
    # ...
```

## Tips & Tricks

- **Override Precedence**: When you `extend` a service, any value you define in the "child" service **overwrites** the value from the parent. For lists (like `ports` or `volumes`), they are **merged**.
- **The -f (Multiple Files) Flag**: You can also combine files at runtime: `docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d`. This is excellent for keeping production-only secrets or resource limits out of your development files.
- **Variable Substitution**: Always use an `.env` file for values that change between environments (DB passwords, image tags). A well-versed admin never hardcodes a version number in a Compose file.

## Summary

Modularising your Docker Compose files is about reducing "Toil." By using `extends` and `include`, you create a maintainable, auditable, and scalable configuration that can grow with your infrastructure without becoming a burden.
