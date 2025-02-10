---
layout: post
title: "DevOps - Kubernetes: Orchestrating Containerized Applications at Scale"
date: 2019-03-14 09:23:12 +01
categories: devops kubernetes
tags: kubernetes orchestration containerization
---

## Intro

**Kubernetes** (K8s) has emerged as the de facto standard for container orchestration, enabling organizations to deploy, scale, and manage containerized applications efficiently. This guide explores **advanced concepts** in Kubernetes, including custom resource definitions, operators, advanced scheduling, and security best practices to help you leverage the full power of Kubernetes in your DevOps practices.

---

## Step 1: Setting Up a Kubernetes Cluster

Before diving into advanced topics, ensure you have a Kubernetes cluster ready.

### **1.1 Install Minikube for Local Development**

```sh
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Start Minikube:

```sh
minikube start --driver=docker
```

### **1.2 Install kubectl**

```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Verify the installation:

```sh
kubectl version --client
```

---

## Step 2: Deploying Applications with Advanced Configurations

### **2.1 Create a Deployment with Resource Limits**

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          resources:
            limits:
              cpu: "500m"
              memory: "128Mi"
            requests:
              cpu: "250m"
              memory: "64Mi"
```

Apply the deployment:

```sh
kubectl apply -f nginx-deployment.yaml
```

### **2.2 Implement Horizontal Pod Autoscaling**

Create an HPA for the nginx deployment:

```yml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        targetAverageUtilization: 50
```

Apply the HPA:

```sh
kubectl apply -f nginx-hpa.yaml
```

---

## Step 3: Advanced Scheduling Techniques

### **3.1 Node Affinity**

Schedule pods on specific nodes based on labels:

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: nginx
      image: nginx
```

### **3.2 Pod Topology Spread Constraints**

Distribute pods across nodes or zones:

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-spread
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: nginx
  containers:
    - name: nginx
      image: nginx
```

---

## Step 4: Custom Resource Definitions (CRDs) and Operators

### **4.1 Create a Custom Resource Definition**

Define a custom resource for managing databases:

```yml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                engine:
                  type: string
                version:
                  type: string
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
      - db
```

Apply the CRD:

```sh
kubectl apply -f database-crd.yaml
```

### **4.2 Implement a Kubernetes Operator**

Create a simple operator using the Operator SDK:

```sh
operator-sdk init --domain example.com --repo github.com/example/database-operator
operator-sdk create api --group example --version v1 --kind Database --resource --controller
```

Implement the controller logic in `controllers/database_controller.go`.

---

## Step 5: Advanced Networking

### **5.1 Network Policies**

Implement network segmentation:

```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

### **5.2 Service Mesh with Istio**

Install Istio:

```sh
istioctl install --set profile=demo -y
```

Enable Istio injection for a namespace:

```sh
kubectl label namespace default istio-injection=enabled
```

---

## Step 6: Security Best Practices

### **6.1 Pod Security Policies**

Enforce security standards for pods:

```yml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  volumes:
    - "configMap"
    - "emptyDir"
    - "projected"
    - "secret"
    - "downwardAPI"
```

### **6.2 RBAC (Role-Based Access Control)**

Create a role and role binding:

```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Step 7: Monitoring and Logging

### **7.1 Prometheus for Monitoring**

Deploy Prometheus using Helm:

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus
```

### **7.2 EFK Stack for Logging**

Deploy the Elasticsearch, Fluentd, and Kibana (EFK) stack:

```sh
kubectl create namespace logging
helm repo add elastic https://helm.elastic.co
helm install elasticsearch elastic/elasticsearch -n logging
helm install kibana elastic/kibana -n logging
kubectl apply -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-elasticsearch.yaml
```

---

## Conclusion

Kubernetes provides a powerful platform for orchestrating containerized applications at scale. By leveraging advanced features like custom resources, operators, advanced scheduling, and implementing security best practices, you can build robust, scalable, and secure applications.
