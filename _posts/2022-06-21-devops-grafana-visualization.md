---
layout: post
title: "DevOps - Grafana: Implementing Dynamic 'Multi-Variable' Dashboards for Fleet Management"
date: 2022-06-21 09:49:35 +01
categories: devops monitoring
tags: grafana monitoring visualisation devops dashbaords
---

## The Problem: Dashboard Sprawl

You have 10 different clusters, each with 50 nodes. If you create a static dashboard for every host, you end up with 500 dashboards. When you want to add a new "Disk Latency" panel, you have to do it 500 times. This is the definition of "toil" that a smart DevOps engineer should avoid.

## The Optimal Solution: Chained Query Variables

Variables allow you to create one "Master" dashboard that updates its panels based on dropdown selections. The real power comes when you "chain" variables (e.g., selecting a 'Cluster' updates the 'Host' list).

### Step 1: Create the 'Cluster' Variable

1.  **Dashboard Settings -> Variables -> New**.
2.  **Type**: Query.
3.  **Query**: `label_values(node_uname_info, cluster_name)`

### Step 2: Create the 'Host' Variable (Chained)

1.  **Type**: Query.
2.  **Query**: `label_values(node_uname_info{cluster_name="$cluster"}, instance)`
    _Note: The `$cluster` variable from step 1 is used to filter the host list._

### Step 3: Use the Variables in Panels

In your PromQL queries, replace hardcoded values with the variables:

```promql
rate(node_cpu_seconds_total{instance="$host", mode!="idle"}[5m])
```

## Tips & Tricks

- **The 'All' Value**: Always enable the "Include All option" in variable settings. In your query, use the regex operator `=~` instead of `=`:
  ```promql
  node_memory_MemAvailable_bytes{instance=~"$host"}
  ```
- **Ad-Hoc Filters**: Add a variable of type "Ad-hoc filters." This adds a button to the top of the dashboard that allows you to filter the _entire_ dashboard by any label (e.g., `env=prod`) on the fly, without pre-defining it.
- **Variable Regex**: If your hostnames are messy (e.g., `web-01.prod.internal:9100`), use a regex in the variable configuration to strip the port or domain for a cleaner dropdown: `/([^.]+)/`.

## Summary

Mastering Grafana variables is about creating "Infrastructure as Code" for your visualisations. A single, well-designed dynamic dashboard can replace hundreds of static ones, making your monitoring stack more maintainable and much more useful for the rest of your team.
