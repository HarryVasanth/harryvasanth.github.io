---
layout: post
title: "Linux - Optimize MySQL Performance for High-Traffic Websites"
date: 2018-05-15 14:30:22 +01
categories: linux hosting
tags: hosting mysql
---

## Intro

Out of the box, MySQL is a well-polished relational database, but high-traffic websites can strain its performance if not optimized properly. This guide explores advanced techniques to optimize MySQL for handling heavy workloads, focusing on query optimization, indexing strategies, configuration tuning, and hardware considerations.

---

## Step 1: Optimize Queries and Schema Design

### **1.1 Use EXPLAIN for Query Analysis**

The `EXPLAIN` statement helps analyze how MySQL executes queries. It identifies inefficiencies like full table scans.

#### Example:

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 123 AND order_date > '2025-01-01';
```

Check the output for:

- **Type**: Prefer `index` or `const` over `ALL`.
- **Rows**: Lower is better.

### **1.2 Avoid SELECT \***

Fetching unnecessary columns increases I/O. Specify only required columns:

```sql
SELECT order_id, order_date FROM orders WHERE customer_id = 123;
```

### **1.3 Normalize and Denormalize Appropriately**

- Normalize to reduce data redundancy.
- Denormalize selectively for read-heavy workloads to avoid expensive JOINs.

---

## Step 2: Indexing Strategies

### **2.1 Use Composite Indexes**

For queries with multiple conditions, create composite indexes:

```sql
ALTER TABLE orders ADD INDEX (customer_id, order_date);
```

This index benefits queries filtering by both `customer_id` and `order_date`.

### **2.2 Avoid Over-Indexing**

Too many indexes slow down write operations. Regularly review unused indexes:

```sql
SHOW INDEX FROM orders;
```

### **2.3 Leverage Covering Indexes**

A covering index contains all columns needed by a query:

```sql
ALTER TABLE orders ADD INDEX (customer_id, order_date, total_amount);
```

This avoids accessing the table directly.

---

## Step 3: Configuration Tuning

### **3.1 Adjust Key Buffer Size**

The key buffer stores index blocks for MyISAM tables. Increase it for faster lookups:

```conf
[mysqld]
key_buffer_size = 256M
```

### **3.2 Optimize InnoDB Buffer Pool**

For InnoDB tables, the buffer pool caches data and indexes. Allocate ~70% of system memory:

```conf
[mysqld]
innodb_buffer_pool_size = 8G
```

### **3.3 Tune Query Cache**

Enable query caching for read-heavy workloads:

```conf
[mysqld]
query_cache_size = 128M
query_cache_type = 1
```

Disable it for write-heavy systems as it may cause contention.

---

## Step 4: Partitioning and Sharding

### **4.1 Table Partitioning**

Partition large tables to improve query performance:

```sql
CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    order_date DATE,
    ...
) PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p0 VALUES LESS THAN (2020),
    PARTITION p1 VALUES LESS THAN (2025),
    PARTITION p2 VALUES LESS THAN MAXVALUE
);
```

### **4.2 Database Sharding**

Distribute data across multiple databases based on a shard key (e.g., `customer_id`). Use application logic or middleware like ProxySQL to route queries.

---

## Step 5: Monitor and Analyze Performance

### **5.1 Enable Slow Query Log**

Log queries that exceed a threshold execution time:

```conf
[mysqld]
slow_query_log = 1
long_query_time = 2
slow_query_log_file = /var/log/mysql-slow.log
```

Analyze the log using tools like `pt-query-digest`.

### **5.2 Use Performance Schema**

Enable the Performance Schema to monitor resource usage and bottlenecks:

```conf
[mysqld]
performance_schema = ON
```

---

## Step 6: Hardware Optimization

### **6.1 Use SSDs**

Switch to SSDs for faster disk I/O, especially for random reads/writes.

### **6.2 Scale Vertically**

Upgrade CPU and RAM to handle higher concurrency and larger datasets.

### **6.3 Scale Horizontally with Replication**

Set up master-slave replication to distribute read traffic:

```conf
[mysqld]
server-id = 1 # Master ID
log_bin = /var/log/mysql-bin.log # Enable binary logging

# On Slave(s):
server-id = 2 # Unique ID for each slave
replicate-do-db = your_database_name # Optional: replicate specific DBs only.
```

---

## Conclusion

Regularly monitor performance metrics and adapt optimizations and respective hardware components as traffic patterns evolve.
