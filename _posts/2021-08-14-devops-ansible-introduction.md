---
layout: post
title: "DevOps - Ansible: Using 'delegate_to' and 'run_once' for Multi-Tier Orchestration"
date: 2021-08-14 17:53:21 +01
categories: devops automation
tags: ansible orchestration devops automation performance
---

## Beyond Simple Playbooks

Most Ansible tutorials show you how to install a package on a group of servers. In a highly skilled infrastructure role, you often need to orchestrate actions _across_ different tiers. For example, you need to tell your Load Balancer to stop sending traffic to `web01` before you begin the update on `web01`.

## The Tool: delegate_to

`delegate_to` allows a task to execute on a machine other than the current `inventory_hostname`.

### Scenario: Zero-Downtime Deployment

We want to update 10 web servers, one by one, ensuring each is removed from the HAProxy load balancer first.

```yaml
- hosts: webservers
  serial: 1 # Crucial: Process one server at a time
  tasks:
    - name: Disable server in HAProxy
      community.network.haproxy:
        state: disabled
        host: "{{ inventory_hostname }}"
        backend: app_pool
      delegate_to: lb01 # This runs on the load balancer, NOT the web server

    - name: Update Web Application
      apt:
        name: my-app
        state: latest

    - name: Enable server in HAProxy
      community.network.haproxy:
        state: enabled
        host: "{{ inventory_hostname }}"
        backend: app_pool
      delegate_to: lb01
```

{: file='rolling_deploy.yml'}

## The Tool: run_once

Sometimes you have a task that only needs to happen once for the entire play, regardless of how many hosts are in the inventory (e.g., running a database migration).

```yaml
- name: Run DB Migrations
  command: /usr/bin/php artisan migrate
  run_once: true
```

## Troubleshooting Key Considerations

- **Variable Scope**: When you use `delegate_to`, the variables available are still those of the _original_ host, not the delegated host. If you need the delegated host's IP, you must access it via `hostvars['lb01']['ansible_default_ipv4']['address']`.
- **Connection Overhead**: If you delegate 100 tasks to a single master server, Ansible will open 100 separate SSH connections. Use `pipelining = True` in your `ansible.cfg` to mitigate this.
- **Local Delegation**: `delegate_to: localhost` is frequently used to interact with cloud APIs (AWS/Azure) or to send a Slack notification after a deployment is complete.

## Summary

Advanced orchestration is about managing dependencies between hosts. By mastering `delegate_to` and `run_once`, you move from being a "server configurer" to an "infrastructure orchestrator."
