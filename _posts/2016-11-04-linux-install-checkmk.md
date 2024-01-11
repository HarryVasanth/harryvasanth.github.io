---
layout: post
title: "Linux - Install Checkmk"
date: 2016-11-04 11:20:09 +01
categories: linux
tags: checkmk
---

## Introduction

Checkmk is a powerful open-source monitoring solution that allows you to keep track of the health and performance of your IT infrastructure. In this guide, we'll explore advanced configurations and best practices to optimize your Checkmk setup.

### Installation

If you haven't installed Checkmk yet, follow these steps to set up a basic instance:

```bash
# Install dependencies
sudo apt update
sudo apt install apache2 libapache2-mod-php php php-xml php-mbstring php-curl

# Download and install Checkmk Raw Edition
wget https://checkmk.com/support/2.0.0p10/check-mk-raw-2.0.0p10_0.buster_amd64.deb
sudo dpkg -i check-mk-raw-2.0.0p10_0.buster_amd64.deb
sudo apt install -f

# Start the Checkmk service
sudo systemctl start apache2
sudo systemctl start xinetd
```

Now that you have a basic setup, let's explore some advanced configurations.

## Advanced Configurations

### 1. Custom Checks

Extend Checkmk's capabilities by creating custom checks for specific applications or services. You can define custom plugins and integrate them into your monitoring setup.

```bash
# Example: Creating a custom check for monitoring a specific application
sudo mkdir /usr/lib/check_mk_agent/local
sudo vim /usr/lib/check_mk_agent/local/my_custom_check

# Add your custom script, e.g., a Python script
#!/usr/bin/env python
def my_custom_check():
    # Your custom check logic here
    pass

# Make the script executable
sudo chmod +x /usr/lib/check_mk_agent/local/my_custom_check
```

### 2. Automation with Checkmk API

Automate tasks and integrate Checkmk into your existing infrastructure using the Checkmk API. This enables you to manage hosts, services, and configurations programmatically.

```bash
# Example: Using Checkmk API to add a new host
curl -X POST -k --user 'your_username:your_password' \
  'https://your_checkmk_server/mysite/check_mk/webapi.py?action=add_host' \
  -d '_username=new_host&_folder=your_folder&_tags=tag1,tag2'
```

### 3. Distributed Monitoring

Scale your monitoring infrastructure by setting up a distributed Checkmk environment. Distribute monitoring tasks across multiple Checkmk instances to optimize resource usage.

```bash
# Example: Configuring distributed monitoring
# Edit /etc/check_mk/main.mk on the master instance
distributed_monitoring_servers += {
    "slave1": {
        "address": "slave1.example.com",
        "socket": "tcp:6557",
    },
}

# Edit /etc/check_mk/main.mk on the slave instance
register_slave("master.example.com")
```

### 4. Agent-Based Monitoring

Agent-based monitoring provides detailed insights into the target systems. Install the Checkmk agent on your monitored servers:

```bash
# Download and install the agent
wget https://checkmk.com/support/2.0.0p13/check-mk-agent-2.0.0p13-1.noarch.rpm
sudo rpm -ivh check-mk-agent-2.0.0p13-1.noarch.rpm

# Enable and start the agent service
sudo systemctl enable --now xinetd
```

### 5. Service Discovery Rules

Create custom service discovery rules to automatically detect and monitor specific services on your hosts. For example, discover all running Docker containers:

```bash
# Create a rule file
sudo vim /etc/check_mk/conf.d/docker_containers.mk

# Add the following content
```

```text
# Discover all Docker containers
inventory_processes += [
  ( ["docker"], ALL_HOSTS, "docker-container", None ),
]
```

Save the file and restart Checkmk:

```bash
sudo systemctl restart apache2
```

### 6. Notifications

Configure notifications to receive alerts when issues occur. Add a notification rule to notify via email:

```bash
# Create a notification rule file
sudo vim /etc/check_mk/conf.d/notify_via_email.mk

# Add the following content
```

```text
# Notify via email
extra_notification_cfg += [
    ( "mail", ALL_HOSTS, [
        ( "your.email@example.com", ["Critical", "Down"], ALL_SERVICES ),
    ] ),
]
```

Save the file and restart Checkmk:

```bash
sudo systemctl restart apache2
```

## Best Practices

### 1. Regular Backups

Perform regular backups of your Checkmk configuration, including custom scripts and plugins. This ensures quick recovery in case of system failure or accidental misconfigurations.

### 2. Monitoring Performance

Optimize your Checkmk instance for better performance. Adjust polling intervals, implement Livestatus caching, and fine-tune database settings based on your monitoring requirements.

### 3. Notifications

Configure notifications effectively to receive alerts promptly. Integrate Checkmk with email, SMS, or third-party services to stay informed about critical issues.

## Conclusion

By implementing these advanced configurations and best practices, you can maximize the capabilities of Checkmk and ensure a robust monitoring solution for your IT infrastructure.
