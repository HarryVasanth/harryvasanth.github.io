---
layout: post
title: "DevOps - Implement Continuous Integration with GitLab CI/CD and Docker"
date: 2019-02-18 09:15:22 +01
categories: devops gitlab
tags: docker gitlab ci-cd
---

## Intro

Implementing Continuous Integration and Continuous Deployment (CI/CD) with GitLab and Docker provides a powerful, flexible, and scalable solution for automating software development workflows. This guide explores **advanced concepts** in GitLab CI/CD with Docker, including multi-stage pipelines, custom Docker images, caching strategies, and advanced deployment techniques to help you build robust CI/CD pipelines.

---

## Step 1: Setting Up GitLab Runner with Docker

GitLab Runner executes your CI/CD jobs. We'll set it up to use Docker.

### **1.1 Install GitLab Runner**

```sh
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner
```

### **1.2 Register Runner with Docker Executor**

```sh
sudo gitlab-runner register \
  --url https://gitlab.com/ \
  --registration-token YOUR_REGISTRATION_TOKEN \
  --executor docker \
  --docker-image docker:stable
```

---

## Step 2: Creating a Multi-Stage Pipeline

Multi-stage pipelines allow you to define complex workflows with dependencies.

### **2.1 Example .gitlab-ci.yml**

```yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG

build:
  stage: build
  image: docker:stable
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE

test:
  stage: test
  image: $DOCKER_IMAGE
  script:
    - npm install
    - npm test

deploy:
  stage: deploy
  image: docker:stable
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker pull $DOCKER_IMAGE
    - docker run -d -p 80:80 $DOCKER_IMAGE
  only:
    - master
```

This pipeline builds a Docker image, runs tests, and deploys the application.

---

## Step 3: Custom Docker Images for CI/CD

Create specialized Docker images for your CI/CD pipeline to improve performance and consistency.

### **3.1 Dockerfile for CI/CD**

```dockerfile
FROM node:14

# Install dependencies
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Install global npm packages
RUN npm install -g mocha

# Set working directory
WORKDIR /app

# Copy package.json and install dependencies
COPY package.json .
RUN npm install

# Copy the rest of the application
COPY . .

# Run tests
CMD ["npm", "test"]
```

### **3.2 Use Custom Image in .gitlab-ci.yml**

```yml
test:
  stage: test
  image: $CI_REGISTRY_IMAGE/ci-image:latest
  script:
    - npm test
```

---

## Step 4: Caching Strategies

Implement caching to speed up your pipeline by reusing data between jobs and pipelines.

### **4.1 Cache npm Dependencies**

```yml
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/

build:
  stage: build
  script:
    - npm ci
  artifacts:
    paths:
      - node_modules/
```

### **4.2 Cache Docker Images**

Use GitLab's Container Registry to store and reuse Docker images:

```yml
build:
  stage: build
  image: docker:stable
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker pull $CI_REGISTRY_IMAGE:latest || true
    - docker build --cache-from $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
```

---

## Step 5: Advanced Deployment Techniques

Implement advanced deployment strategies like blue-green deployments or canary releases.

### **5.1 Blue-Green Deployment**

```yml
deploy:
  stage: deploy
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker pull $DOCKER_IMAGE
    - docker stop blue_app || true
    - docker rm blue_app || true
    - docker run -d --name green_app -p 8080:80 $DOCKER_IMAGE
    - sleep 10
    - if curl http://localhost:8080 | grep "Welcome"; then
      docker stop blue_app || true;
      docker rm blue_app || true;
      docker rename green_app blue_app;
      docker run -d --name green_app -p 80:80 $DOCKER_IMAGE;
      else
      docker stop green_app;
      docker rm green_app;
      exit 1;
      fi
```

This script implements a basic blue-green deployment strategy, ensuring zero-downtime deployments.

---

## Step 6: Dynamic Environments

Create dynamic environments for each branch or merge request.

### **6.1 Dynamic Environment Configuration**

```yml
deploy_review:
  stage: deploy
  script:
    - echo "Deploy to $CI_ENVIRONMENT_SLUG"
    - kubectl create namespace $CI_ENVIRONMENT_SLUG
    - kubectl apply -f k8s/ -n $CI_ENVIRONMENT_SLUG
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: http://$CI_ENVIRONMENT_SLUG.example.com
    on_stop: stop_review
  only:
    - branches
  except:
    - master

stop_review:
  stage: deploy
  variables:
    GIT_STRATEGY: none
  script:
    - kubectl delete namespace $CI_ENVIRONMENT_SLUG
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  when: manual
  only:
    - branches
  except:
    - master
```

This configuration creates a new environment for each branch and provides a manual way to stop and remove the environment.

---

## Step 7: Security Scanning

Integrate security scanning into your CI/CD pipeline.

### **7.1 Container Scanning**

```yml
container_scanning:
  stage: test
  image:
    name: docker.io/aquasec/trivy:latest
    entrypoint: [""]
  variables:
    DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    SEVERITY: "HIGH,CRITICAL"
  script:
    - trivy image --severity $SEVERITY --no-progress --exit-code 1 --quiet $DOCKER_IMAGE
  only:
    - master
```

This job scans your Docker image for vulnerabilities using Trivy.

---

## Conclusion

Implementing advanced CI/CD pipelines with GitLab and Docker enables powerful, flexible, and secure software delivery workflows. By leveraging multi-stage pipelines, custom Docker images, caching strategies, and advanced deployment techniques, you can create efficient, reliable, and scalable CI/CD processes tailored to your project's needs.
