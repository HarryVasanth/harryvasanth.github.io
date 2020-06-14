---
layout: post
title: "Docker - Advanced Dockerfiles: Multi-Stage Builds and Security Hardening in Production"
date: 2020-06-15 00:46:00 +01
categories: docker devops
tags: docker security devops development best-practices automation
---

## The Anatomy of a Production-Grade Dockerfile

In the world of professional infrastructure administration, a Dockerfile that merely "works" is insufficient. A production-ready image must be fast to build, minimal in size, and secure by default. Achieving this requires mastering two core architectural patterns: **Multi-Stage builds** and **Non-Root execution contexts**. This guide explores the implementation details that separate a development-only image from a production-hardened artifact in 2020.

## Why 'Bloat' is a Security Risk

A standard Docker image often contains a full operating system, package managers, compilers, and debugging tools. This "bloat" isn't just a storage issue; it represents a significant increase in the attack surface. If an attacker gains access to your running container, a pre-installed `curl`, `git`, or `gcc` becomes a weapon they can use to download malware or compile exploits to escape the container.

## Stage 1: The Build Environment

The first stage of our Dockerfile includes everything needed to compile the application. This stage will be large, but it remains strictly on your build server or CI/CD runner. It never gets pushed to your production registry.

```dockerfile
# STAGE 1: THE COMPILER
FROM golang:1.14-alpine AS builder

# Set the working directory
WORKDIR /app

# Install essential build dependencies
# We use --no-cache to keep the builder stage as clean as possible
RUN apk add --no-cache git build-base

# Copy only the dependency definition files first
# This is the 'Secret' to fast builds: if these files don't change,
# Docker will skip the 'go mod download' step in subsequent builds.
COPY go.mod go.sum ./
RUN go mod download

# Copy the actual source code
COPY . .

# Perform the build
# -ldflags="-w -s" removes the symbol table and debug information,
# significantly reducing the size of the resulting binary.
RUN go build -ldflags="-w -s" -o /app/server
```

## Stage 2: The Runtime Environment

The second stage starts from a fresh, minimal base image (like `alpine` or the even smaller `scratch`). We copy _only_ the final compiled binary from the builder stage.

```dockerfile
# STAGE 2: THE RUNTIME
FROM alpine:3.12

# Good practice: Add metadata for traceability
LABEL maintainer="admin@example.com"
LABEL version="1.0"

WORKDIR /root/

# Security Pillar: The Non-Root User
# Most containers run as root by default. If an attacker escapes the app,
# they have root access to the host's kernel namespaces.
RUN adduser -D -g '' appuser

# Copy ONLY the artifact from the first stage
COPY --from=builder /app/server .

# Set correct ownership and permissions
RUN chown appuser:appuser /root/server && chmod 500 /root/server

# Switch to the non-privileged user
USER appuser

# Document the port
EXPOSE 8080

# Start the application
CMD ["./server"]
```

## Optimising the Build Context

A common mistake that leads to slow builds is a massive **Build Context**. When you run `docker build .`, the entire content of the current directory is sent to the Docker daemon. If you have a `node_modules` folder, a large dataset, or a `.git` directory with years of history, you are wasting gigabytes of transfer on every build.

**The Fix: .dockerignore**
Create a file named `.dockerignore` in your project root. This functions exactly like `.gitignore`.

```text
.git
node_modules
dist
*.log
docker-compose.yml
Dockerfile
README.md
tests/
```

## Advanced Hardening: Read-Only Filesystems

Once you have a non-root user, you can take security a step further by running the container with a **Read-Only Filesystem**. This prevents an attacker from writing a malicious script to `/tmp` or modifying any part of your application at runtime.

In your `docker-compose.yml` or Kubernetes manifest:

```yaml
services:
  myapp:
    image: myapp:latest
    read_only: true
    tmpfs:
      - /tmp
      - /run
```

## Summary: The Professional Standard

High-quality Docker images are the result of intentional architectural choices. By separating your build and runtime environments and strictly enforcing non-root execution, you create a deployment artifact that is optimised for both speed and security.

- **Fast**: Small images pull faster across the network, reducing deployment windows.
- **Lean**: Minimal images consume less disk space in your registry and on your hosts.
- **Secure**: Removing tools and running as a non-root user makes your infrastructure a significantly harder target.

This should be the non-negotiable standard for every production workload in a modern infrastructure environment. It is the cornerstone of a mature container strategy in 2020.
