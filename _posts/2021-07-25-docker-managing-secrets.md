---
layout: post
title: "Docker - Swarm Secrets: Zero-Downtime Rotation and Versioning"
date: 2021-07-25 06:30:42 +01
categories: docker security
tags: docker-swarm secrets security devops automation
---

## The Immutable Secret Problem

Docker Secrets are designed to be "Write Once, Read Never" (except by the container). Once a secret is created, it is immutable. You cannot edit it. This creates a challenge when a password expires or an API key is leaked. If you delete the secret and recreate it, any service using it will fail to start because it's looking for an ID that no longer exists.

## The Optimal Solution: Versioned Rotation

The professional way to handle this is to treat secrets as versioned entities and use Docker's rolling update mechanism to switch between them.

### Step 1: Create the New Version

```bash
# Original secret was 'db_password_v1'
echo "new_secure_password" | docker secret create db_password_v2 -
```

### Step 2: Perform the Rolling Update

We use the `--secret-rm` and `--secret-add` flags simultaneously. Crucially, we use the `target` parameter to keep the filename inside the container consistent.

```bash
docker service update \
  --secret-rm db_password_v1 \
  --secret-add source=db_password_v2,target=db_password \
  my-production-app
```

## Why this is 'Bulletproof'

1.  **Consistency**: The application always looks for `/run/secrets/db_password`. It doesn't know (or care) about the versioning happening on the host.
2.  **Availability**: Docker Swarm updates containers one at a time. Half your containers will be on v1, and half on v2, until the update completes. Both passwords must be valid in your database during this window!
3.  **Rollback**: If the new password fails, you can immediately roll back the service to use `db_password_v1` without having to re-type anything.

## Tips & Tricks

- **Automated Cleanup**: Docker doesn't automatically delete old secrets. After a successful rotation, you should prune old versions: `docker secret rm db_password_v1`.
- **External Secret Managers**: For very large environments, consider using **HashiCorp Vault** with a sidecar container or an "init container" pattern to inject secrets, as managing 100+ Docker Secrets manually becomes unmanageable.
- **Permissions**: By default, secrets are mounted as `0444` owned by `root`. If your app runs as a non-root user (e.g., `node` or `www-data`), use the `uid` and `gid` options in your Compose file or `service update` command.

## Summary

Secret rotation is a requirement for compliance (SOC2/ISO27001). Mastering versioned rotation in Docker Swarm ensures you can meet these security standards without impacting the uptime of your applications.
