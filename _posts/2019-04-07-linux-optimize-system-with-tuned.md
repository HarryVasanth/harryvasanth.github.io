---
layout: post
title: "Linux - Optimize System Performance with Tuned"
date: 2019-04-07 11:20:09 +01
categories: linux optimization
tags: linux tuned optimization
---

## Intro

Optimizing system performance is critical for ensuring efficient resource utilization and maintaining reliability under varying workloads. **Tuned** is a dynamic tuning daemon for Linux that simplifies performance optimization by applying predefined or custom profiles tailored to specific use cases. This guide explores **advanced concepts** in using Tuned, including custom profile creation, dynamic tuning, and integration with system components like CPU, memory, and I/O schedulers.

---

## Step 1: Installing and Managing Tuned

### **1.1 Install Tuned**

Ensure Tuned is installed on your system:

```sh
# For Debian-based systems
sudo apt install tuned -y

# For Red Hat-based systems
sudo yum install tuned -y
```

### **1.2 Enable and Start the Service**

Enable and start the Tuned daemon to ensure it runs at boot:

```sh
sudo systemctl enable tuned
sudo systemctl start tuned
```

### **1.3 Verify Installation**

Check the status of the Tuned service:

```sh
sudo systemctl status tuned
```

---

## Step 2: Exploring Predefined Profiles

Tuned provides several predefined profiles optimized for different workloads.

### **2.1 List Available Profiles**

View all available profiles:

```sh
tuned-adm list
```

### **2.2 Apply a Profile**

Activate the `throughput-performance` profile for high throughput workloads:

```sh
tuned-adm profile throughput-performance
```

### **2.3 Verify Active Profile**

Check the currently active profile:

```sh
tuned-adm active
```

### **2.4 Common Profiles Overview**

- `balanced`: General-purpose optimization balancing performance and energy savings.
- `throughput-performance`: Optimized for high I/O throughput.
- `latency-performance`: Reduces latency for time-sensitive workloads.
- `virtual-guest`: Optimized for virtualized environments.

---

## Step 3: Custom Profile Creation

Custom profiles allow fine-tuning of system parameters to meet specific requirements.

### **3.1 Create a Custom Profile**

Create a directory for your custom profile:

```sh
sudo mkdir /etc/tuned/custom-profile
```

Create a configuration file (`tuned.conf`) for the profile:

```sh
sudo nano /etc/tuned/custom-profile/tuned.conf
```

#### Example Configuration:

```conf
[main]
include=throughput-performance

[cpu]
governor=performance

[disk]
readahead=4096

[vm]
transparent_hugepages=never
swappiness=10
```

This profile:

- Sets the CPU governor to `performance`.
- Increases disk read-ahead to 4096 KB.
- Disables transparent hugepages and reduces swappiness to optimize memory usage.

### **3.2 Activate the Custom Profile**

Apply the custom profile:

```sh
tuned-adm profile custom-profile
```

---

## Step 4: Dynamic Tuning with Plugins

Tuned uses plugins to dynamically adjust system parameters based on workload.

### **4.1 CPU Tuning**

Optimize CPU frequency scaling and governor settings:

```conf
[cpu]
governor=performance
energy_perf_bias=performance
min_perf_pct=100
```

This ensures maximum CPU performance by disabling power-saving features.

### **4.2 Disk I/O Optimization**

Optimize disk I/O settings for faster read/write operations:

```conf
[disk]
readahead=4096
elevator=deadline
write_cache=on
```

- Sets read-ahead to 4096 KB.
- Uses the `deadline` scheduler for predictable I/O performance.
- Enables write caching for faster writes.

### **4.3 Network Optimization**

Tune network parameters for low latency or high throughput:

```conf
[net]
tcp_rmem_min=4096
tcp_rmem_default=87380
tcp_rmem_max=16777216
tcp_wmem_min=4096
tcp_wmem_default=16384
tcp_wmem_max=16777216
```

These settings adjust TCP buffer sizes to optimize network throughput.

---

## Step 5: Monitoring and Debugging Tuned

### **5.1 Analyze Profile Settings**

View detailed information about the active profile:

```sh
tuned-adm recommend  # Suggests an optimal profile based on workload.
tuned-adm list       # Lists all available profiles.
tuned-adm active     # Displays the currently active profile.
```

### **5.2 Debugging Mode**

Enable debugging to troubleshoot issues with Tuned:

```sh
sudo tuned -D --debug > tuned-debug.log 2>&1 &
tail -f tuned-debug.log
```

---

## Step 6: Advanced Use Cases

### **6.1 Combining Profiles**

Combine multiple profiles to leverage their benefits:

```conf
[main]
include=throughput-performance,virtual-host

[cpu]
governor=performance

[disk]
elevator=noop
readahead=8192
```

This combines `throughput-performance` and `virtual-host` while overriding specific settings.

### **6.2 Applying Profiles Temporarily**

Apply a profile temporarily without affecting persistent settings:

```sh
tuned-adm profile latency-performance --runtime-only
```

The profile will revert after a reboot.

---

## Step 7: Performance Validation

After applying a profile, validate its impact using monitoring tools:

### **7.1 CPU Usage**

Monitor CPU performance:

```sh
htop      # Interactive process viewer.
lscpu     # Displays CPU architecture details.
mpstat    # Reports CPU usage statistics.
```

### **7.2 Disk Performance**

Test disk I/O performance:

```sh
fio --name=test --rw=read --bs=4k --size=1G --numjobs=4 --runtime=60 --group_reporting
iostat -x     # Displays detailed disk I/O statistics.
```

### **7.3 Network Throughput**

Measure network bandwidth:

```sh
iperf3 -c <server-ip>
netstat -i    # Displays network interface statistics.
tc qdisc show # Shows traffic control configurations.
```

---

## Conclusion

Tuned simplifies Linux system optimization by dynamically adjusting system parameters based on predefined or custom profiles tailored to specific workloads. By leveraging advanced features like custom profiles, dynamic tuning plugins, and integration with key subsystems (CPU, memory, disk I/O, and networking), you can achieve significant performance improvements while maintaining flexibility and control over your environment.

Experiment with different profiles and validate their effects using monitoring tools to ensure optimal results tailored to your workload needs.
