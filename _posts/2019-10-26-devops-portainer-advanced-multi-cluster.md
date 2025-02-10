---
layout: post
title: "DevOps - Portainer: Advanced Multi-Cluster and Secure Container Management"
date: 2019-10-26 15:25:03 +01
categories: devops portainer
tags: portainer docker kubernetes container-management
---

## Intro

Portainer has evolved into a comprehensive container management platform, simplifying the orchestration of Docker, Kubernetes, and other containerized environments. With features like multi-cluster management, advanced role-based access control (RBAC), and API-driven automation, Portainer is an essential tool for DevOps teams. This guide explores **advanced concepts** in Portainer, including multi-cluster setups, secure access control, GitOps workflows, and API automation to help you optimize your container management practices.

---

## Step 1: Setting Up Portainer for Multi-Cluster Management

Portainer enables centralized management of multiple Docker and Kubernetes clusters.

### **1.1 Deploy Portainer**

Deploy Portainer as a Docker container:

```sh
docker volume create portainer_data
docker run -d -p 9000:9000 -p 9443:9443 \
  --name=portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access the Portainer UI at `https://<host-ip>:9443` and set up an admin account.

### **1.2 Add Multiple Environments**

In the Portainer UI:

1. Navigate to **Environments > Add Environment**.
2. Add Docker or Kubernetes clusters by providing their API endpoints.
3. For remote environments, install the **Portainer Agent**:
   ```sh
   docker run -d -p 9001:9001 \
     --name=portainer_agent \
     --restart=always \
     -v /var/run/docker.sock:/var/run/docker.sock \
     portainer/agent
   ```

### **1.3 Manage Multi-Cluster Workloads**

Switch between environments in the UI to manage containers, stacks, and services across clusters.

---

## Step 2: Advanced Role-Based Access Control (RBAC)

Portainer’s RBAC allows you to enforce granular permissions for users and teams.

### **2.1 Create Teams and Users**

1. Navigate to **Users > Teams**.
2. Create teams (e.g., `Dev`, `Ops`) and assign users to them.

### **2.2 Assign Environment Roles**

Define roles for each environment:

- **Administrator**: Full access.
- **Operator**: Limited access to manage workloads.
- **Read-Only**: View-only permissions.

Example configuration:

```yml
Team: Dev
Environment Role: Operator
```

### **2.3 Secure Sensitive Resources**

Restrict access to specific stacks or containers:

1. Go to the resource (e.g., a stack).
2. Click **Access Control** and assign team-specific permissions.

---

## Step 3: Automating Workflows with GitOps

GitOps integrates version control into your deployment workflows.

### **3.1 Enable GitOps in Portainer**

Navigate to **Settings > GitOps**, then configure:

- Repository URL (e.g., `https://github.com/my-org/my-repo.git`).
- Authentication (SSH key or token).

### **3.2 Deploy Applications Using GitOps**

Create a stack linked to a Git repository:

```yml
apiVersion: v1
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
          image: nginx:latest
```

Push changes to the repository, and Portainer will automatically sync and deploy updates.

---

## Step 4: Advanced Monitoring and Troubleshooting

Portainer provides built-in monitoring tools for container health and performance.

### **4.1 Monitor Container Logs**

View logs directly from the Portainer UI:

1. Select a container.
2. Click on **Logs** to view real-time output for debugging.

### **4.2 Resource Utilization**

Monitor CPU and memory usage for containers or nodes under **Environments > Nodes**.

### **4.3 Troubleshooting with Live Console**

Access a live terminal for containers via the UI:

1. Select a container.
2. Click on **Console**, choose a shell (e.g., `bash`), and troubleshoot interactively.

---

## Step 5: API Automation with Portainer

Portainer’s HTTP API allows you to automate tasks programmatically.

### **5.1 Authenticate with the API**

Generate a JWT token for authentication:

```sh
http POST <portainer-url>/api/auth Username="admin" Password="your-password"
```

Use the token in subsequent requests:

```yml
Authorization: Bearer <JWT_TOKEN>
```

### **5.2 List All Containers**

Retrieve all containers in an environment:

```sh
http GET <portainer-url>/api/endpoints/1/docker/containers/json \
X-API-Key:<your-access-token> all==true
```

### **5.3 Create a New Container**

Deploy an NGINX container via the API:

```sh
http POST <portainer-url>/api/endpoints/1/docker/containers/create \
X-API-Key:<your-access-token> \
name=="web01" Image="nginx:latest" \
ExposedPorts='{"80/tcp": {}}' \
HostConfig='{"PortBindings": {"80/tcp": [{"HostPort": "8080"}]}}'
```

Start the container:

```sh
http POST <portainer-url>/api/endpoints/1/docker/containers/<container-id>/start \
X-API-Key:<your-access-token>
```

---

## Step 6: Advanced Security Features

### **6.1 Secure Your Installation**

Enable HTTPS by providing custom certificates during deployment:

```sh
docker run -d -p 9443:9443 \
  --name=portainer-secure \
  --restart=always \
  -v /path/to/cert.pem:/certs/cert.pem \
  -v /path/to/key.pem:/certs/key.pem \
  portainer/portainer-ce --sslcert /certs/cert.pem --sslkey /certs/key.pem
```

### **6.2 Limit API Access**

Restrict API access by IP address using firewalls or reverse proxies like NGINX.

Example NGINX configuration:

```conf
server {
    listen       443 ssl;
    server_name  portainer.example.com;

    ssl_certificate      /etc/nginx/ssl/cert.pem;
    ssl_certificate_key  /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://localhost:9443;
        allow <trusted-ip>;
        deny all;
    }
}
```

---

## Step 7: Scaling with Kubernetes Integration

Portainer simplifies Kubernetes management with guided workflows and multi-cluster support.

### **7.1 Deploy Kubernetes Applications**

Use pre-designed application templates in Portainer’s UI or upload custom manifests.

Example manifest for an NGINX deployment:

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
          image: nginx
          ports:
            - containerPort: 80
```

Apply it via the Kubernetes environment in Portainer.

---

## Conclusion

Portainer empowers DevOps engineers with advanced capabilities for managing multi-cluster environments, automating workflows, securing deployments, and integrating GitOps principles—all through an intuitive interface or its powerful API. By leveraging these advanced features, you can streamline container orchestration across Docker, Kubernetes, and hybrid environments while ensuring security and scalability.
