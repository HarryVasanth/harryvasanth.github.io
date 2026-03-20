---
layout: post
title: "DevOps - Monitoring: Advanced Log Aggregation with Vector and Loki"
date: 2023-11-15 20:54:47 +01
categories: devops monitoring
tags: vector loki grafana devops logging monitoring observability
---

## The 'Logging Tax' Problem

In a large-scale microservice environment, logging often becomes the most expensive part of the infrastructure. Traditional "ELK" (Elasticsearch, Logstash, Kibana) stacks require massive amounts of RAM and disk space because they index every single word in your logs. This is the "Logging Tax."

In 2023, the industry has shifted toward a more efficient model: **Grafana Loki** for storage and **Vector** for processing. Loki is "Prometheus for logs"-it only indexes metadata (labels), not the log content itself, reducing storage costs by up to 90%. Vector is a high-performance, Rust-based data router that can transform and filter logs with minimal CPU overhead.

## The Architecture: The High-Performance Pipeline

1.  **Source**: Applications, Syslog, or Docker logs.
2.  **Collector/Processor (Vector)**: Runs on every host. It cleans the logs, removes sensitive data (PII), and adds metadata (environment, datacenter).
3.  **Storage (Loki)**: A central cluster that stores the compressed log chunks.
4.  **Visualization (Grafana)**: Where you query the logs using LogQL.

## Implementation: Configuring Vector to 'Sanitise' Logs

Vector uses a powerful transformation language called VRL (Vector Remap Language). This allows you to perform production ready data engineering at the edge.

```toml
# /etc/vector/vector.toml
[sources.my_app_logs]
type = "file"
include = ["/var/log/myapp/*.log"]

[transforms.filter_pii]
type = "remap"
inputs = ["my_app_logs"]
source = """
# Redact anything that looks like a credit card number
.message = replace(.message, r'[0-9]{4}-[0-9]{4}-[0-9]{4}-[0-9]{4}', "[REDACTED_CC]")

# Parse JSON if the message starts with {
if starts_with(.message, "{") {
    .parsed, err = parse_json(.message)
    if err == null {
        del(.message)
        ., err = merge(., .parsed)
    }
}
"""

[sinks.loki_cluster]
type = "loki"
inputs = ["filter_pii"]
endpoint = "http://loki:3100"
labels.env = "production"
labels.host = "{{ host }}"
```

## Why This is a Suitable Workflow

- **Cost Management**: By filtering out "Debug" logs at the source using Vector, you never pay to store them in Loki.
- **Performance**: Vector is written in Rust. It can handle 100k+ events per second on a single CPU core, whereas Logstash (Java) would require gigabytes of RAM for the same load.
- **Schema-on-Query**: Like Prometheus, Loki allows you to parse the log content _at the time of the query_. This gives you the flexibility to change your mind about what data is important without re-indexing your entire history.

## Querying the Truth: LogQL

With Loki, you can perform powerful "Metrics from Logs" queries:

```logql
# Calculate the requests per second for 5xx errors from the 'frontend' service
sum(rate({app="frontend"} |= "500" [5m]))
```

This bridges the gap between monitoring (Prometheus) and logging (Loki), allowing you to see _that_ a spike happened and _why_ it happened in the same dashboard.

## Summary

Observability in 2023 is about efficiency and speed. By replacing heavy, expensive legacy stacks with the Vector and Loki combination, you provide your team with high-resolution visibility into application behavior without breaking the infrastructure budget. It is the definitive logging architecture for the modern, cost-conscious DevOps engineer.
