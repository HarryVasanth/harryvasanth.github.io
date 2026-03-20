---
layout: post
title: "DevOps - Kubernetes: Mastering Resource Management with Quotas and LimitRanges"
date: 2020-03-15 16:30:17 +01
categories: devops kubernetes
tags: kubernetes k8s devops resource-management governance linux
---

## The Chaos of the Ungoverned Cluster

In the early stages of Kubernetes adoption, the focus is often on simply getting applications to run. However, as more teams move to a shared production cluster, a "Tragedy of the Commons" occurs. A single microservice with a memory leak or an infinite loop in a Java application can consume all available resources on a node, triggering the OOM (Out of Memory) killer and starving critical system components like the Kubelet or API server.

A smart DevOps engineer knows that stability is not accidental. It requires strict resource governance using **ResourceQuotas** and **LimitRanges**. This guide explains how to enforce fiscal and technical responsibility within your namespaces in a 2020 production environment.

## 1. ResourceQuotas: The Hard Ceiling

ResourceQuotas are applied at the namespace level. They limit the _total_ amount of resources that all pods in that namespace can consume collectively.

### Implementation: The 'Production' Guardrail

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: production-api
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
```

With this quota, the API team can deploy as many pods as they like, but they cannot exceed 8 physical CPU cores in total. This prevents one team from accidentally (or intentionally) monopolising the entire cluster budget.

## 2. LimitRanges: The Standardised Pod

While quotas limit the namespace, **LimitRanges** limit the individual pod or container. They provide a "Safety Net" by ensuring that every container has a default resource request and limit, even if the developer forgets to include them in their deployment manifest.

### Implementation: Preventing 'Runaway' Containers

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-mem-limit-range
spec:
  limits:
    - default: # The limit used if none is specified
        cpu: 500m
        memory: 512Mi
      defaultRequest: # The request used if none is specified
        cpu: 200m
        memory: 256Mi
      max: # The maximum any single container can request
        cpu: "2"
        memory: 4Gi
      type: Container
```

**Why this is critical**: If a container starts without limits, it is classified as "Best Effort" by the Kubernetes scheduler. In a resource crunch, "Best Effort" pods are the first to be killed. By enforcing a `defaultRequest`, you ensure your pods are properly placed on nodes with enough head-room.

## 3. The 'Requests' vs. 'Limits' Paradox

Understanding the difference between these two is the key to cluster density:

- **Requests**: What the scheduler uses to find a node. If you request 2GB, the scheduler ensures the node has 2GB free.
- **Limits**: What the kernel (cgroups) uses to throttle the container. If you exceed your CPU limit, you are throttled. If you exceed your memory limit, you are killed (OOMKilled).

**Tip**: For critical production services, set **Requests equal to Limits**. This places the pod in the "Guaranteed" Quality of Service (QoS) class, making it the most stable and least likely to be evicted during a node failure.

## 4. Monitoring Resource Pressure

Applying quotas is only half the battle. You must monitor the "Pressure" within your namespaces using Prometheus and Grafana.

- **CLI**: `kubectl describe quota -n production-api`
- **Prometheus**: Use the `kube_resourcequota` metric to trigger an alert when a team is at 90% of their limit. "Hey Team API, you are about to hit your quota. Do you need a budget increase or are your apps leaking?"

## 5. Horizontal Pod Autoscaler (HPA) Integration

Once limits are in place, you can safely use the HPA. The HPA relies on metrics like CPU percentage. If you haven't defined a CPU _request_, the HPA cannot calculate the percentage and will fail to scale your application during a traffic spike.

## Summary: Governance as an Enabler

Resource governance is often viewed as a restriction, but in a professional environment, it is an enabler. It provides the mathematical certainty that one team's failure won't cause a global outage. By mastering ResourceQuotas and LimitRanges, you transform a fragile, unpredictable cluster into a stable, multi-tenant platform. This is the hallmark of a mature Kubernetes operations strategy. It ensures that the "noisy neighbour" problem is a thing of the past and that your infrastructure remains cost-effective and resilient in 2020.
