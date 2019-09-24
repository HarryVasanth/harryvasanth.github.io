---
layout: post
title: "DevOps - Kubernetes: Advanced Pod Scheduling and Resource Management"
date: 2019-09-24 14:50:37 +01
categories: devops kubernetes
tags: kubernetes pod-scheduling resource-management
---

## Intro

Kubernetes offers powerful mechanisms for advanced pod scheduling and resource management, allowing administrators to optimize cluster utilization and application performance. This guide explores **advanced concepts** in Kubernetes scheduling and resource management, including node affinity, pod affinity/anti-affinity, taints and tolerations, resource quotas, and limit ranges to help you implement sophisticated scheduling strategies.

---

## Step 1: Node Affinity

Node affinity allows you to constrain which nodes your pod can be scheduled on based on node labels.

### **1.1 Required Node Affinity**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b
  containers:
    - name: nginx
      image: nginx
```

This example ensures the pod is scheduled only on nodes labeled with `zone=us-east-1a` or `zone=us-east-1b`.

### **1.2 Preferred Node Affinity**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-preferred-affinity
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: disk-type
                operator: In
                values:
                  - ssd
  containers:
    - name: nginx
      image: nginx
```

This configuration prefers nodes with SSD storage but doesn't require it.

---

## Step 2: Pod Affinity and Anti-Affinity

Pod affinity and anti-affinity allow you to schedule pods based on the labels of pods already running on nodes.

### **2.1 Pod Affinity**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - cache
          topologyKey: kubernetes.io/hostname
  containers:
    - name: web-app
      image: nginx
```

This pod will be scheduled on a node that is already running a pod with the label `app=cache`.

### **2.2 Pod Anti-Affinity**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: redis-cache
  labels:
    app: store
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - store
          topologyKey: "kubernetes.io/hostname"
  containers:
    - name: redis-server
      image: redis:3.2-alpine
```

This configuration ensures that no two pods with the label `app=store` are scheduled on the same node.

---

## Step 3: Taints and Tolerations

Taints and tolerations work together to ensure that pods are not scheduled onto inappropriate nodes.

### **3.1 Adding a Taint to a Node**

```sh
kubectl taint nodes node1 key=value:NoSchedule
```

### **3.2 Pod with Toleration**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-toleration
spec:
  tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx
```

This pod can be scheduled on the tainted node.

---

## Step 4: Resource Quotas

Resource quotas provide constraints that limit aggregate resource consumption per namespace.

### **4.1 Creating a Resource Quota**

```yml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "10"
```

This quota limits the total CPU, memory, and number of pods in a namespace.

### **4.2 Applying Quota to a Namespace**

```sh
kubectl apply -f resource-quota.yaml --namespace=myapp
```

---

## Step 5: Limit Ranges

Limit ranges enforce minimum and maximum compute resources usage per pod or container in a namespace.

### **5.1 Creating a Limit Range**

```yml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-mem-cpu-per-container
spec:
  limits:
    - default:
        cpu: 100m
        memory: 512Mi
      defaultRequest:
        cpu: 50m
        memory: 256Mi
      type: Container
```

This limit range sets default resource requests and limits for containers in the namespace.

### **5.2 Applying Limit Range to a Namespace**

```sh
kubectl apply -f limit-range.yaml --namespace=myapp
```

---

## Step 6: Advanced Scheduling Techniques

### **6.1 Combining Node Affinity and Pod Anti-Affinity**

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: zone
                    operator: In
                    values:
                      - us-east-1a
                      - us-east-1b
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - web
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx
```

This deployment ensures pods are scheduled in specific zones while preferring to spread them across different nodes.

### **6.2 Using Taints for Dedicated Nodes**

```sh
# Taint nodes for a specific workload
kubectl taint nodes node1 dedicated=ml:NoSchedule
kubectl taint nodes node2 dedicated=ml:NoSchedule
```

```yml
# Deploy pods that tolerate the taint
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-workload
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-app
  template:
    metadata:
      labels:
        app: ml-app
    spec:
      tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "ml"
          effect: "NoSchedule"
      containers:
        - name: ml-container
          image: ml-image:latest
```

This setup dedicates specific nodes to machine learning workloads.

---

## Conclusion

Advanced pod scheduling and resource management in Kubernetes provide powerful tools for optimizing cluster utilization and application performance. By leveraging node affinity, pod affinity/anti-affinity, taints and tolerations, resource quotas, and limit ranges, you can create sophisticated scheduling strategies tailored to your specific workload requirements. These techniques enable better resource allocation, improved application reliability, and enhanced cluster efficiency.
