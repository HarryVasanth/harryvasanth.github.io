---
layout: post
title: "Docker - Deploy a Flask App with Docker Compose"
date: 2025-01-10 18:19:44 +01
categories: docker
tags: flask hosting advanced
---

## Intro

This guide expands on deploying a Flask app using Docker Compose, incorporating advanced features and best practices for production-ready applications. By the end, you'll have a robust, scalable, and secure deployment pipeline.

---

## Step 1: Create a Flask App

Start by creating a directory for your project:

```bash
mkdir flask-docker && cd flask-docker
```

Inside, create `app.py`:

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return "Hello, Docker!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Add Templates and Static Files (Optional)

For dynamic content, create a `templates` directory with an HTML file:

```html
<!-- templates/home.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Flask App</title>
  </head>
  <body>
    <h1>Welcome to Flask with Docker!</h1>
  </body>
</html>
```

Update `app.py` to render the template:

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route('/')
def home():
    return render_template('home.html')
```

---

## Step 2: Write an Optimized Dockerfile

Create a `Dockerfile`:

```dockerfile
# Use a lightweight base image
FROM python:3.11-slim

# Set environment variables for production
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Set working directory
WORKDIR /app

# Install dependencies in one layer to optimize caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Expose the application port
EXPOSE 5000

# Run the application
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "app:app"]
```

### Best Practices for Dockerfile:

1. **Use Multi-Stage Builds** for smaller images:

   ```dockerfile
   FROM python:3.11-slim AS builder
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt

   FROM python:3.11-slim AS final
   WORKDIR /app
   COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
   COPY . .
   CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "app:app"]
   ```

2. **Minimize Layers** by combining commands.

---

## Step 3: Create an Advanced docker-compose.yml File

Define services and advanced configurations:

```yml
version: "3.8"

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
    environment:
      FLASK_ENV: production
    depends_on:
      - redis
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5000 || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  redis_data:
```

### Key Features Explained:

- **Health Checks** ensure services are ready before others start.
- **Custom Networks** isolate communication between services.
- **Volumes** persist data across container restarts.

---

## Step 4: Build and Run the Application

Build the Docker image:

```bash
docker compose build --no-cache
```

Run the containers in detached mode:

```bash
docker compose up -d
```

Check service health:

```bash
docker ps --filter "health=healthy"
```

Access the app at `http://localhost:5000`.

---

## Step 5: Add Multi-Environment Support

For different environments (e.g., development, production), use multiple Compose files:

### Example Setup:

1. `docker-compose.yml` (Base Configuration)
2. `docker-compose.override.yml` (Development Overrides)
3. `docker-compose.prod.yml` (Production Overrides)

Run with specific configurations:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Step 6: Secure and Optimize Your Deployment

### Security Best Practices:

1. Use non-root users in containers.
2. Limit resources to prevent overuse:
   ```yml
   deploy:
     resources:
       limits:
         cpus: "1"
         memory: "512M"
   ```
3. Scan images for vulnerabilities using tools like `trivy`.

### Performance Optimization:

- Enable caching layers in Dockerfile.
- Use lightweight base images like `alpine`.
- Optimize database queries and indexing.

---

## Advanced Features to Explore

### Logging and Monitoring:

Integrate tools like ELK Stack or Prometheus/Grafana for centralized logging and monitoring.

### CI/CD Pipelines:

Automate builds and deployments using GitHub Actions or GitLab CI/CD.

### Scaling with Load Balancers:

Use tools like Traefik or Nginx to distribute traffic across multiple containers.

---

## Conclusion

Congratulations! You've deployed a production-ready Flask app using Docker Compose with advanced configurations and best practices. This setup is scalable, secure, and ready for real-world applications.
