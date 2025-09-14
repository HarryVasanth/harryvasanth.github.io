---
layout: post
title: "DevOps - ArgoCD: Implementing 'Self-Healing' and 'Pruning' for a Declarative Cluster"
date: 2025-09-14 17:41:44 +01
categories: devops kubernetes
tags: argocd gitops kubernetes devops automation srg
---

## The Problem: Configuration Drift in Kubernetes

You have a perfect Kubernetes setup defined in Git. But on a Friday night, an engineer uses `kubectl edit` to manually increase a memory limit or change a container image to fix a bug. They forget to update Git. Your "Source of Truth" is now a lie. This is **Configuration Drift**, and it's the leading cause of "it worked in staging but failed in production."

**ArgoCD** is the industry standard for GitOps. It doesn't just "deploy" your code; it continuously monitors your cluster and compares it to your Git repository.

## The Advanced Architecture: The 'Application' Resource

In ArgoCD, you define an `Application` manifest. This tells ArgoCD where the Git repo is and which cluster to target.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: billing-api
spec:
  project: default
  source:
    repoURL: "https://github.com/org/billing-api.git"
    targetRevision: HEAD
    path: k8s/production
  destination:
    server: "https://kubernetes.default.svc"
    namespace: billing
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Phase 2: Mastering Self-Healing and Pruning

These are the "Power Features" that ensure your cluster remains identical to your code:

1.  **Self-Healing**: If anyone manually changes a resource via `kubectl`, ArgoCD will detect the difference and **instantly overwrite** it with the version from Git.
2.  **Pruning**: If you delete a YAML file from Git, ArgoCD will **delete** the corresponding resource in Kubernetes. Without this, your cluster becomes a "Graveyard" of old, unused resources.

## Phase 3: Sync Waves and Hooks

In a complex application, you can't just deploy everything at once. You need the Database to be ready before the API starts.
**ArgoCD Sync Waves** allow you to order the deployment.

```yaml
# In the Database Manifest
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"

# In the API Manifest
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "10"
```

ArgoCD will wait for everything in Wave 1 to be healthy before starting Wave 10.

## Phase 4: The 'Diff' as a Debugging Tool

ArgoCD provides a powerful visual "Diff." Before you sync, you can see exactly what will change. This is the ultimate safety check for a experienced administrator. "Why is Terraform trying to recreate the LoadBalancer?" The Diff will show you that a single character changed in the annotation.

## Summary

GitOps is not just about using Git; it's about making Git the **only** way to change the cluster. By implementing ArgoCD with Self-Healing and Pruning, you move from "Managing Servers" to "Managing Intent." Your cluster becomes predictable, auditable, and incredibly easy to recover in a disaster. This is the definitive standard for any professional Kubernetes deployment in 2025.
