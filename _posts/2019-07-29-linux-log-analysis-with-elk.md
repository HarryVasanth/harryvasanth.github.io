---
layout: post
title: "Linux - Implement Log Analysis with ELK Stack (Elasticsearch, Logstash, Kibana)"
date: 2019-07-29 13:10:42 +01
categories: linux logs
tags: elk-stack log-analysis
---

## Intro

The **ELK Stack**—comprising Elasticsearch, Logstash, and Kibana—is a powerful suite for managing, analyzing, and visualizing log data. It provides a scalable solution for centralizing logs from multiple sources, parsing and enriching them, and creating actionable insights through visualizations. This guide explores **advanced concepts** in implementing the ELK stack for log analysis, including multi-source ingestion, custom pipelines, index management, and security best practices to help you build a robust log analysis system.

---

## Step 1: Setting Up the ELK Stack

### **1.1 Install Elasticsearch**

Install Elasticsearch on your server:

```sh
sudo apt update
sudo apt install elasticsearch -y
```

Configure `elasticsearch.yml` to bind Elasticsearch to localhost or a specific IP:

```yml
network.host: localhost
```

Start and enable the Elasticsearch service:

```sh
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
```

Verify installation:

```sh
curl -X GET "localhost:9200"
```

### **1.2 Install Logstash**

Install Logstash:

```sh
sudo apt install logstash -y
```

Create a basic configuration file (`/etc/logstash/conf.d/logstash.conf`):

```conf
input {
  file {
    path => "/var/log/syslog"
    start_position => "beginning"
  }
}
filter {
  grok {
    match => { "message" => "%{SYSLOGLINE}" }
  }
}
output {
  elasticsearch {
    hosts => ["localhost:9200"]
  }
}
```

Start Logstash:

```sh
sudo systemctl start logstash
```

### **1.3 Install Kibana**

Install Kibana:

```sh
sudo apt install kibana -y
```

Configure `kibana.yml` to point to Elasticsearch:

```yml
server.port: 5601
elasticsearch.hosts: ["http://localhost:9200"]
```

Start Kibana:

```sh
sudo systemctl start kibana
sudo systemctl enable kibana
```

Access Kibana at `http://<your-server-ip>:5601`.

---

## Step 2: Advanced Log Ingestion with Logstash

### **2.1 Multi-Source Ingestion**

Ingest logs from multiple sources such as files, syslog, and APIs:

```conf
input {
  file {
    path => ["/var/log/nginx/access.log", "/var/log/nginx/error.log"]
    start_position => "beginning"
  }
  syslog {
    port => 514
  }
  http_poller {
    urls => {
      api_logs => "http://example.com/api/logs"
    }
    schedule => { every => "5m" }
  }
}
```

### **2.2 Enrich Logs with Filters**

Use filters to parse and enrich logs with additional metadata.

#### Example: Parsing Nginx Logs with Grok

```conf
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
}
```

#### Example: GeoIP Enrichment for IP Addresses

```conf
filter {
  geoip {
    source => "clientip"
  }
}
```

---

## Step 3: Index Management in Elasticsearch

### **3.1 Custom Index Naming**

Send logs to custom indices based on their source:

```conf
output {
  if [source] == "/var/log/nginx/access.log" {
    elasticsearch {
      index => "nginx-access-%{+YYYY.MM.dd}"
      hosts => ["localhost:9200"]
    }
  } else if [source] == "/var/log/syslog" {
    elasticsearch {
      index => "syslog-%{+YYYY.MM.dd}"
      hosts => ["localhost:9200"]
    }
  }
}
```

### **3.2 Index Lifecycle Management (ILM)**

Set up ILM policies to manage index retention:

```conf
{
  "policy": {
    "phases": {
      "hot": { "actions": { "rollover": { "max_size": "50GB", "max_age": "30d" } } },
      "delete": { "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}
}
```

Apply the policy to an index template:

```sh
curl -X PUT "localhost:9200/_index_template/nginx-template" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["nginx-*"],
  "template": { ... },
  "policy_name": "nginx-policy"
}'
```

---

## Step 4: Visualizing Data in Kibana

### **4.1 Create Index Patterns**

In Kibana, go to **Management > Index Patterns** and create patterns for your indices (e.g., `nginx-*`).

### **4.2 Build Dashboards**

Use Kibana’s visualization tools to create dashboards:

- Add histograms for request counts.
- Use pie charts for error distribution.
- Create maps for geographic data using GeoIP-enriched fields.

---

## Step 5: Securing the ELK Stack

### **5.1 Enable Authentication**

Enable Elasticsearch’s built-in authentication (requires X-Pack):

```sh
bin/elasticsearch-setup-passwords interactive
```

Set passwords for `elastic`, `kibana`, and other users.

Update `kibana.yml` with credentials:

```yml
elasticsearch.username: kibana_system
elasticsearch.password: <password>
```

### **5.2 Restrict Access**

Restrict external access to Elasticsearch by binding it to localhost or using a reverse proxy like Nginx.

#### Example Nginx Configuration:

```conf
server {
    listen       80;
    server_name  example.com;

    location / {
        proxy_pass http://localhost:5601;
        auth_basic           "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

Generate `.htpasswd`:

```sh
htpasswd -c /etc/nginx/.htpasswd user1
```

---

## Step 6: Monitoring and Scaling the ELK Stack

### **6.1 Monitor Performance**

Use Kibana’s Stack Monitoring feature to monitor Elasticsearch nodes and resource usage.

### **6.2 Scale Elasticsearch**

Add more nodes to your cluster for better performance:

```yml
cluster.name: my-cluster
node.name: node-2
network.host: <node-ip>
discovery.seed_hosts: ["<master-node-ip>"]
cluster.initial_master_nodes: ["<master-node-name>"]
```

Restart the new node and verify cluster health:

```sh
curl -X GET "localhost:9200/_cluster/health?pretty"
```

---

## Conclusion

The ELK Stack provides a comprehensive solution for centralized log analysis, enabling you to ingest logs from multiple sources, enrich data with filters, manage indices effectively, and visualize insights through Kibana dashboards. By following these advanced techniques—including custom pipelines, index lifecycle management, and security best practices—you can build a scalable and secure logging architecture tailored to your needs.
