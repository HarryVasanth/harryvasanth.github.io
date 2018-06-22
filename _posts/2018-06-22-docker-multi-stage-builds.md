---
layout: post
title: "Docker - Implement Multi-Stage Builds for Efficient Images"
date: 2018-06-22 09:45:17 +01
categories: docker
tags: docker optimization
---

## Intro

Multi-stage builds in Docker allow you to create smaller, more efficient container images by separating build and runtime environments. This approach reduces image size, improves security, and simplifies maintenance. In this guide, we’ll explore advanced multi-stage build concepts focusing on real-world scenarios.

---

## Step 1: Basics of Multi-Stage Builds

A multi-stage build uses multiple `FROM` instructions in a single Dockerfile. Each stage can inherit artifacts from the previous one, allowing you to include only necessary components in the final image.

### Key Advantages:

- Smaller image sizes.
- Reduced attack surface by excluding build tools.
- Cleaner and more maintainable Dockerfiles.

---

## Step 2: Python Example - Lightweight Production Image

Imagine a Python application with dependencies that require compilation. Using multi-stage builds, we can separate the build environment (with compilers) from the runtime environment.

### **Dockerfile**

```dockerfile
# Stage 1: Build environment
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build tools and dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

RUN pip install --no-cache-dir --upgrade pip && \
    pip wheel --no-cache-dir --no-deps --wheel-dir /wheels -r requirements.txt

# Stage 2: Runtime environment
FROM python:3.11-slim AS runtime

WORKDIR /app

# Copy only necessary files from the builder stage
COPY --from=builder /wheels /wheels
COPY requirements.txt .

# Install prebuilt wheels (no compilers needed)
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir --find-links=/wheels -r requirements.txt

COPY . .

CMD ["python", "main.py"]
```

### Key Concepts:

1. **Builder Stage**:

   - Installs compilers (e.g., `gcc`) and builds Python wheels.
   - Outputs compiled dependencies into `/wheels`.

2. **Runtime Stage**:
   - Excludes compilers, using only prebuilt wheels for installation.
   - Copies application code and runs it in a lightweight environment.

---

## Step 3: Bash Example - Dynamic Build Scripts

For applications requiring dynamic setup during the build process, you can use Bash scripts to orchestrate tasks.

### **Scenario**:

A Python application requires a dataset generated during the build process.

### **start.sh**

```sh
#!/bin/bash
echo "Generating dataset..."
python generate_data.py

echo "Starting application..."
python main.py
```

### **Dockerfile**

```dockerfile
# Stage 1: Build dataset
FROM python:3.11-slim AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN python generate_data.py

# Stage 2: Runtime environment
FROM python:3.11-slim AS runtime

WORKDIR /app

COPY --from=builder /app/dataset.csv ./dataset.csv
COPY . .

CMD ["bash", "start.sh"]
```

### Key Concepts:

1. **Build Dataset**:

   - The `generate_data.py` script creates a dataset (`dataset.csv`).
   - The dataset is copied to the runtime stage.

2. **Dynamic Start Script**:
   - The `start.sh` script ensures flexibility for runtime operations.

---

## Step 4: Advanced Techniques for Optimization

### **4.1 Conditional Dependencies**

Use build arguments to conditionally include development or production dependencies.

```dockerfile
ARG ENV=production

RUN if [ "$ENV" = "development" ]; then \
      pip install debugpy; \
    fi
```

### **4.2 Multi-Platform Builds**

Leverage Docker's BuildKit to create images for multiple architectures (e.g., AMD64, ARM64).

```sh
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t my-app .
```

---

## Step 5: Testing During Build Process

You can integrate testing into your multi-stage builds to ensure image reliability.

```dockerfile
# Stage 1: Build and test environment
FROM python:3.11-slim AS tester

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN pytest tests/

# Stage 2: Runtime environment
FROM python:3.11-slim AS runtime

WORKDIR /app

COPY . .

CMD ["python", "main.py"]
```

If the tests fail during the `pytest` step, the image build process will stop, ensuring only tested code reaches production.

---

## Step 6: Reducing Image Size with Scratch Base Image (Bash Example)

For minimal environments, use a `scratch` base image with statically compiled binaries.

### **Dockerfile**

```dockerfile
# Stage 1: Compile binary using Bash script
FROM debian:buster AS builder

WORKDIR /app

COPY script.sh .
RUN chmod +x script.sh && ./script.sh > output.txt

# Stage 2: Minimal runtime image
FROM scratch AS runtime

COPY --from=builder /app/output.txt /output.txt

CMD ["cat", "/output.txt"]
```

This approach is ideal for lightweight applications that don't require an OS layer.

---

## Conclusion

By effectively employing multi-stage builds, you can reduce image sizes, improve security, and streamline your CI/CD pipelines.
