---
layout: post
title: "Docker - Secure Docker Containers: Best Practices and Tools"
date: 2019-06-13 08:55:18 +01
categories: docker
tags: docker security
---

## Intro

Securing Docker containers is essential for protecting your applications and infrastructure from vulnerabilities and attacks. Docker's lightweight architecture introduces unique security challenges, such as kernel sharing and image integrity. This guide explores **advanced concepts** in Docker container security, including best practices, tools for scanning and runtime protection, and to secure your Docker environment effectively.

---

## Step 1: Secure Docker Images

### **1.1 Use Minimal Base Images**

Minimize the attack surface by using lightweight base images like `alpine`:

```dockerfile
FROM alpine:3.18
RUN apk add --no-cache python3 py3-pip
```

Avoid bloated images that include unnecessary software, such as `ubuntu` or `debian`, unless required for your application.

### **1.2 Pin Image Versions**

Always specify the exact version of the base image to avoid unexpected updates:

```dockerfile
FROM nginx:1.23.4
```

### **1.3 Scan Images for Vulnerabilities**

Use tools like **Trivy** or **Anchore** to detect vulnerabilities in your images:

```sh
trivy image nginx:1.23.4
```

Example output:

```sh
CVE-2023-12345  High  OpenSSL  Upgrade to version 1.2.3
```

### **1.4 Multi-Stage Builds**

Use multi-stage builds to keep sensitive data out of the final image:

```dockerfile
# Stage 1: Build
FROM golang:1.20 AS builder
WORKDIR /app
COPY . .
RUN go build -o app .

# Stage 2: Final Image
FROM alpine:3.18
WORKDIR /app
COPY --from=builder /app/app .
CMD ["./app"]
```

This approach ensures that build tools and intermediate files are not included in the final image.

---

## Step 2: Secure Container Runtime

### **2.1 Run Containers as Non-Root Users**

Avoid running containers as root to minimize privilege escalation risks:

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
CMD ["node", "app.js"]
```

### **2.2 Drop Unnecessary Capabilities**

Restrict container privileges by dropping capabilities:

```sh
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx:1.23.4
```

### **2.3 Use Read-Only Filesystems**

Prevent modification of container files by enabling read-only mode:

```sh
docker run --read-only nginx:1.23.4
```

### **2.4 Limit Resource Usage**

Set CPU and memory limits to prevent resource exhaustion:

```sh
docker run --memory=512m --cpus=1 nginx:1.23.4
```

---

## Step 3: Secure Networking

### **3.1 Avoid Exposing Unnecessary Ports**

Expose only required ports and bind them to specific interfaces:

```sh
docker run -p 127.0.0.1:8080:80 nginx:1.23.4
```

### **3.2 Use Network Segmentation**

Isolate containers using Docker networks:

```sh
docker network create --driver bridge isolated_net
docker run --net=isolated_net nginx:1.23.4
```

### **3.3 Enable TLS for Communication**

Use TLS to secure communication between the Docker client and daemon:

```json
{
  "tls": true,
  "tlscert": "/path/to/cert.pem",
  "tlskey": "/path/to/key.pem"
}
```

---

## Step 4: Manage Secrets Securely

### **4.1 Use Docker Secrets**

Store sensitive data securely using Docker secrets (requires Swarm mode):

```sh
echo "my-secret-password" | docker secret create db_password -
docker service create \
  --name my-app \
  --secret db_password \
  my-app-image
```

Access the secret within the container at `/run/secrets/db_password`.

### **4.2 Avoid Hardcoding Secrets**

Use `.dockerignore` to exclude sensitive files from builds:

```sh
.env
private.key
config.json
```

---

## Step 5: Monitor and Audit Containers

### **5.1 Monitor Container Activity**

Use tools like Sysdig or Falco to monitor runtime behavior:

```sh
falco --config=/etc/falco/falco.yaml
```

Falco can detect suspicious activity, such as unauthorized file access or privilege escalation attempts.

### **5.2 Collect Logs**

Centralize container logs using a logging driver like Fluentd or AWS CloudWatch:

```json
{
  "log-driver": "fluentd",
  "log-opts": {
    "fluentd-address": "localhost:24224"
  }
}
```

---

## Step 6: Automate Security Scans in CI/CD Pipelines

Integrate image scanning into your CI/CD pipeline using tools like Trivy or Snyk.

#### Example GitLab CI/CD Pipeline with Trivy:

```yml
stages:
  - scan

scan_image:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image my-app-image:${CI_COMMIT_SHA}
```

This scans the built image for vulnerabilities before deployment.

---

## Step 7: Use Advanced Container Security Tools

Leverage specialized tools for comprehensive container security:

| Tool    | Use Case                 | Features                                                            |
| ------- | ------------------------ | ------------------------------------------------------------------- |
| Trivy   | Vulnerability Scanning   | Scans images, filesystems, and Git repositories for vulnerabilities |
| Falco   | Runtime Security         | Detects suspicious activity in containers                           |
| Anchore | Image Compliance         | Validates images against custom policies                            |
| Calico  | Network Security         | Enforces network policies for Kubernetes and Docker                 |
| Snyk    | Vulnerability Management | Identifies vulnerabilities in dependencies and container images     |

---

## Conclusion

Securing Docker containers requires a multi-faceted approach that includes minimizing attack surfaces, managing secrets securely, enforcing runtime restrictions, and monitoring activity for anomalies. By combining best practices with powerful tools like Trivy, Falco, and Anchore, you can build a robust security strategy tailored to your containerized applications.

Start implementing these practices today to protect your infrastructure from potential threats while maintaining performance and scalability.
