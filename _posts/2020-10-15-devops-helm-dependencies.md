---
layout: post
title: "DevOps - Helm: Advanced Dependency Management and Chart Versioning"
date: 2020-10-15 08:30:24 +01
categories: devops kubernetes
tags: helm kubernetes k8s devops automation deployment
---

## The Complexity of Modular Kubernetes Applications

In a professional DevOps environment, we rarely deploy a single application in isolation. A modern microservice often requires a database (PostgreSQL), a cache (Redis), and a message broker (RabbitMQ). Managing these components manually across different environments (Staging, Production) using raw Kubernetes YAML is a recipe for configuration drift and deployment failure.

**Helm**, the package manager for Kubernetes, solves this through the use of **Dependencies**. By nesting charts, you can define your entire stack in a single `Chart.yaml`, ensuring that every developer in your organisation is running the exact same versions of all supporting services.

## 1. Defining Dependencies in Chart.yaml

In Helm v3, dependencies are defined directly in the `Chart.yaml` file. This replaces the old `requirements.yaml` used in v2.

```yaml
apiVersion: v2
name: my-app-stack
version: 1.0.0
dependencies:
  - name: postgresql
    version: 10.x.x
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: 14.x.x
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

## 2. Managing the Chart Lock File

When you run `helm dependency update`, Helm downloads the specified charts and creates a `Chart.lock` file. This file is critical for a experienced administrator.

- **The Golden Rule**: Always commit the `Chart.lock` file to Git.
- **Why?**: It ensures that your CI/CD pipeline uses the exact same SHA-256 hash of the dependency chart that you tested on your local machine. Without this, a sudden update from a chart maintainer could break your production build.

## 3. Value Overriding: The 'Global' Strategy

The most powerful feature of Helm dependencies is the ability to pass configurations down from the "Parent" chart to the "Children."

In your `values.yaml`:

```yaml
# Configure the child postgresql chart
postgresql:
  enabled: true
  auth:
    database: my_prod_db
    username: admin

# Global values can be seen by ALL dependent charts
global:
  imageRegistry: my-private-registry.com
  storageClass: premium-ssd
```

## 4. Building Your Own Reusable Library Charts

A smart DevOps team often creates a "Common Library Chart." This chart doesn't deploy anything itself; instead, it contains reusable templates for standard features like `Ingress`, `ServiceMonitor`, and `NetworkPolicies`.

Your application charts then list the `common-library` as a dependency. This ensures that every team's Ingress follows the company's security standards (e.g., forcing TLS 1.2, adding standard headers).

## 5. Suitable Strategy: Helm-file and GitOps

While Helm is powerful, managing 50 different `helm install` commands is toil. Professional teams use **Helmfile** or **ArgoCD** to manage their Helm releases declaratively.

```yaml
# helmfile.yaml
releases:
  - name: billing-api
    chart: ./charts/my-app-stack
    values:
      - values/production.yaml
```

## Summary: Governance Through Automation

Dependency management in Helm is about more than just "installing apps." It is about providing a standardised, version-controlled platform for your engineering teams. By mastering sub-charts, global values, and the `Chart.lock` mechanism, you create a deployment process that is repeatable, auditable, and resilient to external changes. In 2020, Helm is the definitive standard for packaging Kubernetes infrastructure, and understanding its modular architecture is a non-negotiable skill for any smart DevOps engineer.
