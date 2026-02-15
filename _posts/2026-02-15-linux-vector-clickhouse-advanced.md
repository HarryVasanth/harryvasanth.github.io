---
layout: post
title: "Linux - Logging: High-Performance Real-Time Analytics with Vector and ClickHouse"
date: 2026-02-15 18:00:52 +01
categories: linux monitoring
tags: vector clickhouse logging observability performance linux srg
---

## The End of the 'Expensive' Logging Era

For years, infrastructure teams have struggled with the "Logging Tax"-the massive operational and financial cost of storing and querying billions of system logs. We moved from the heavy ELK stack to the leaner Loki architecture, which helped with storage costs. However, in 2026, we face a new challenge: **Real-Time Analytics**. We no longer just want to "search" for a log line; we want to perform complex SQL queries over terabytes of data to identify trends, security anomalies, and performance regressions in sub-seconds.

The ideal solution to this is the combination of **Vector** (the data router) and **ClickHouse** (the ultra-fast columnar database). ClickHouse is so efficient that it can process millions of rows per second on a single server, making it the perfect analytical engine for high-volume telemetry.

## The Architecture: The High-Speed Streaming Pipeline

1.  **Source (Vector Agent)**: Runs on every host or as a Kubernetes daemonset. It collects logs from Journald, Syslog, or Docker sockets using zero-copy mechanisms where possible.
2.  **Transform (Vector Aggregator)**: Standardises the logs into a structured JSON format. It parses user-agents, adds GeoIP metadata, and redacts sensitive data (PII) at the edge before the data ever leaves your network.
3.  **Sink (ClickHouse)**: Stores the data in a compressed, columnar format. ClickHouse is designed for "Append-Only" workloads, making it perfect for logs.

## Implementation: The ClickHouse Schema for Infrastructure Logs

Unlike schema-less systems, ClickHouse requires a defined structure. This is the secret to its incredible speed. By defining types, ClickHouse can apply aggressive compression algorithms specific to that data type.

```sql
CREATE TABLE logs (
    timestamp DateTime,
    hostname LowCardinality(String),
    service LowCardinality(String),
    level Enum8('info' = 1, 'warn' = 2, 'error' = 3),
    message String,
    request_id String,
    latency_ms UInt32,
    status_code UInt16
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, service, hostname);
```

- **LowCardinality**: A production ready optimisation that reduces the disk space required for repetitive strings like hostnames and service names.
- **Enum8**: Uses a single byte to represent log levels, significantly improving query speed and reducing storage.
- **Partitioning**: Allows for rapid data deletion (TTL) and speeds up queries that are restricted to specific time ranges.

## Configuring Vector for ClickHouse Batching

Batching is critical when writing to ClickHouse. You should never send one log at a time; you should send thousands in a single block to minimize the overhead of the MergeTree engine.

```toml
# /etc/vector/vector.toml
[sources.journal_logs]
type = "journald"

[transforms.json_parsing]
type = "remap"
inputs = ["journal_logs"]
source = """
.parsed, err = parse_json(.message)
if err == null {
    del(.message)
    . = merge(., .parsed)
}
"""

[sinks.clickhouse_output]
type = "clickhouse"
inputs = ["json_parsing"]
endpoint = "http://clickhouse-internal:8123"
database = "logging"
table = "logs"
skip_unknown_fields = true

[sinks.clickhouse_output.batch]
max_bytes = 10485760 # 10MB
timeout_secs = 5
```

## Why This Wins in 2026: Sub-Second SQL

You can query your logs using standard SQL, bridging the gap between infrastructure engineers and data analysts.

```sql
-- Find the average latency per service in the last 15 minutes
SELECT service, avg(latency_ms) as avg_lat FROM logs
WHERE timestamp > now() - interval 15 minute
GROUP BY service
ORDER BY avg_lat DESC;
```

- **Massive Compression**: ClickHouse's columnar storage can compress log data by 10x to 30x compared to standard text files or Elasticsearch.
- **Unified Observability**: You can store logs, metrics, and even trace IDs in the same ClickHouse cluster, providing a single "Source of Truth" for all your infrastructure intelligence.

## Summary: Moving from Archive to Intelligence

In 2026, logging is no longer a "passive archive" that you only check when things break; it is an active stream of intelligence. By leveraging the speed of Vector and the analytical power of ClickHouse, you provide your organisation with the ability to find "needles in haystacks" in real-time. You move beyond simple keyword searches and into the era of true, high-resolution infrastructure observability. This is the definitive architecture for any experienced administrator managing a high-scale, modern environment where data is the most valuable asset.
