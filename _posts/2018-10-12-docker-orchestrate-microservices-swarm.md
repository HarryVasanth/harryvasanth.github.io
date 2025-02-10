---
layout: post
title: "Docker - Orchestrate Microservices with Docker Swarm"
date: 2018-10-12 08:15:28 +01
categories: docker
tags: docker swarm orchestration microservices
---

## Intro

Docker Swarm is a native container orchestration tool that simplifies the deployment, scaling, and management of microservices across multiple nodes. This guide explores **advanced concepts** in Docker Swarm, including service discovery, load balancing, overlay networking, and scaling to help you orchestrate microservices effectively.

---

## Step 1: Setting Up a Docker Swarm Cluster

### **1.1 Initialize the Swarm**

Choose one node as the manager and initialize the swarm:

```sh
docker swarm init --advertise-addr <MANAGER-IP>
```

This command outputs a token for worker nodes to join the swarm.

### **1.2 Add Worker Nodes**

On each worker node, run:

```sh
docker swarm join --token <TOKEN> <MANAGER-IP>:2377
```

### **1.3 Verify the Cluster**

Check the cluster status:

```sh
docker node ls
```

---

## Step 2: Deploying Microservices

### **2.1 Create a Service**

Deploy a Python-based microservice:

```sh
docker service create \
  --name python-app \
  --replicas 3 \
  --publish 5000:5000 \
  python:3.11-slim python -m http.server 5000
```

### **2.2 Inspect the Service**

View service details:

```sh
docker service inspect python-app --pretty
```

### **2.3 Scale the Service**

Scale the service to handle more traffic:

```sh
docker service scale python-app=5
```

The swarm manager automatically schedules additional replicas across available nodes.

---

## Step 3: Networking in Docker Swarm

### **3.1 Create an Overlay Network**

Overlay networks enable communication between services across nodes:

```sh
docker network create \
  --driver overlay \
  my-overlay-network
```

### **3.2 Attach Services to the Network**

Deploy two services (e.g., Python and Bash) on the overlay network:

```sh
docker service create \
  --name python-service \
  --network my-overlay-network \
  python:3.11-slim python -m http.server 5000

docker service create \
  --name bash-service \
  --network my-overlay-network \
  bash sleep infinity
```

### **3.3 Test Connectivity**

Connect to the Bash container and ping the Python service using its DNS name:

```sh
docker exec -it $(docker ps -q -f name=bash-service) bash
ping python-service
```

This demonstrates Docker Swarm's built-in DNS-based service discovery.

---

## Step 4: Load Balancing and High Availability

Docker Swarm provides built-in load balancing to distribute traffic across replicas.

### **4.1 Built-In Load Balancing**

Access `python-app` via its published port (e.g., `http://<MANAGER-IP>:5000`). Traffic is evenly distributed among replicas.

### **4.2 Fault Tolerance**

Simulate a node failure by shutting down a worker node:

```sh
docker node update --availability drain <NODE-ID>
```

Swarm automatically reschedules tasks from the drained node to healthy nodes.

---

## Step 5: Deploying a Multi-Service Stack

Use a `docker-compose.yml` file to deploy a stack of interconnected services.

### **5.1 Example Compose File**

```yml
version: "3.9"
services:
  frontend:
    image: nginx:latest
    ports:
      - "80:80"
    networks:
      - app-network

  backend:
    image: python:3.11-slim
    command: python -m http.server 8000
    networks:
      - app-network

networks:
  app-network:
    driver: overlay
```

### **5.2 Deploy the Stack**

Deploy the stack using Docker Swarm:

```sh
docker stack deploy -c docker-compose.yml my-stack
```

### **5.3 Verify Stack Deployment**

List all running services in the stack:

```sh
docker stack services my-stack
```

---

## Step 6: Monitoring and Debugging

### **6.1 Monitor Services**

Use `docker service ps` to monitor tasks for a specific service:

```sh
docker service ps python-app
```

### **6.2 Debugging Containers**

Access logs for troubleshooting:

```sh
docker service logs python-app --follow
```

---

## Advanced Concepts

### **7.1 Rolling Updates**

Update services without downtime:

```sh
docker service update \
  --image python:3.12-slim \
  python-app
```

Swarm performs rolling updates by replacing replicas incrementally.

### **7.2 Constraints and Placement Preferences**

Control where services run using constraints:

```sh
docker service create \
  --name constrained-service \
  --constraint "node.labels.role==frontend" \
  nginx:latest
```

This ensures that the service only runs on nodes labeled as `frontend`.

Add labels to nodes with:

```sh
docker node update --label-add role=frontend <NODE-ID>
```

---

## Conclusion

Docker Swarm simplifies microservice orchestration by providing features like service discovery, load balancing, fault tolerance, and rolling updates out of the box. By following this guide, you can deploy scalable and resilient microservices using Docker Swarm while leveraging advanced concepts like overlay networking and multi-service stacks.
