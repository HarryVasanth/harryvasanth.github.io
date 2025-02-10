---
layout: post
title: "Proxmox - Implement Storage Replication for Disaster Recovery"
date: 2019-05-20 16:45:33 +01
categories: proxmox
tags: proxmox-ve storage-replication disaster-recovery
---

## Intro

Disaster recovery is a critical component of maintaining business continuity in virtualized environments. **Proxmox VE** offers built-in storage replication features to ensure data redundancy and minimize downtime in the event of hardware or site failures. This guide explores **advanced concepts** in Proxmox storage replication, including ZFS-based replication, stretched clusters, and best practices for disaster recovery to help you implement a robust and efficient strategy.

---

## Step 1: Understanding Proxmox Storage Replication

Proxmox storage replication is a feature that uses **ZFS snapshots** to replicate virtual machine (VM) data between nodes in a cluster. It ensures minimal data loss by synchronizing changes at regular intervals and is particularly useful for disaster recovery scenarios.

### **Key Features**

- **Asynchronous Replication**: Data is replicated periodically, reducing the impact on performance.
- **ZFS Snapshots**: Uses incremental snapshots for efficient data transfer.
- **Failover Support**: Enables quick recovery by promoting replicated VMs on secondary nodes.

---

## Step 2: Preparing Your Environment

### **2.1 Requirements**

1. A Proxmox cluster with at least two nodes.
2. ZFS storage configured on all nodes.
3. Sufficient network bandwidth for replication.

### **2.2 Configure ZFS Storage**

Ensure ZFS is installed and a pool is created on each node:

```sh
zpool create -f rpool /dev/sdX
```

Verify the pool:

```sh
zpool status
```

Add the ZFS pool to Proxmox via the web interface under **Datacenter > Storage > Add > ZFS**.

---

## Step 3: Setting Up Storage Replication

### **3.1 Enable Replication**

1. Navigate to **Datacenter > Replication** in the Proxmox web interface.
2. Add a new replication job:
   - Select the source VM.
   - Choose the target node.
   - Set the replication interval (e.g., every 5 minutes).

#### Example Command-Line Configuration:

Use `pvesr` to configure replication:

```sh
pvesr create local-zfs remote-node remote-zfs --rate 100M --interval 300
```

This replicates data from `local-zfs` to `remote-node`'s `remote-zfs` every 5 minutes with a bandwidth limit of 100 MB/s.

### **3.2 Verify Replication Status**

Check the status of replication jobs:

```sh
pvesr status
```

---

## Step 4: Stretched Clusters for Disaster Recovery

A stretched cluster spans multiple sites, ensuring high availability (HA) within and across locations.

### **4.1 Configure a Stretched Cluster**

1. Set up two Proxmox clusters (e.g., one per site).
2. Use **LINSTOR** with DRBD® to replicate volumes across sites:

   - Install LINSTOR on all nodes:
     ```sh
     apt install linstor-controller linstor-satellite linstor-client drbd-utils
     ```
   - Configure DRBD resources for synchronous replication.

3. Add both clusters to the same Proxmox environment and configure shared storage using LINSTOR.

### **4.2 Benefits of Stretched Clusters**

- Reduces Recovery Point Objective (RPO) to seconds.
- Ensures VMs can fail over seamlessly between sites.

---

## Step 5: Testing Disaster Recovery Scenarios

### **5.1 Simulate Node Failure**

1. Power off the primary node hosting a replicated VM:
   ```sh
   poweroff
   ```
2. Promote the replicated VM on the secondary node:
   ```sh
   qm start <VMID> --node <secondary-node>
   ```

### **5.2 Restore from Backup**

If replication fails, restore from backups stored on a Proxmox Backup Server (PBS):

```sh
proxmox-backup-client restore vm/<VMID> --repository <PBS-URL>
```

---

## Step 6: Best Practices for Storage Replication

### **6.1 Optimize Bandwidth Usage**

Limit replication bandwidth during peak hours:

```sh
pvesr set <job-id> --rate 50M
```

### **6.2 Monitor Replication Jobs**

Set up alerts for failed replication jobs using Prometheus or email notifications:

```sh
cat /var/log/pve/tasks/replication.log | grep ERROR
```

### **6.3 Combine with Backups**

Replication complements but does not replace backups; use PBS for periodic backups:

- Local backups for fast recovery.
- Remote backups for site-wide disaster recovery.

---

## Step 7: Advanced Techniques

### **7.1 Snapshot Management**

Use ZFS snapshots to roll back VMs in case of corruption or accidental changes:

```sh
zfs snapshot rpool/data@backup-20230520
zfs rollback rpool/data@backup-20230520
```

### **7.2 Geographical Redundancy**

Replicate VMs across geographically distributed locations using cloud providers or secondary data centers.

#### Example with Offsite Backup:

Use tools like `rclone` to sync backups to cloud storage:

```sh
rclone sync /mnt/proxmox-backups remote:/backups/proxmox
```

---

## Step 8: Monitoring and Troubleshooting

### **8.1 Check Logs for Errors**

Review logs to diagnose issues:

```sh
journalctl -u pvesr.service
```

### **8.2 Validate Data Integrity**

Verify ZFS pool health regularly:

```sh
zpool scrub rpool
zpool status rpool
```

---

## Conclusion

Implementing storage replication in Proxmox VE enhances disaster recovery capabilities by ensuring data redundancy and minimizing downtime during failures. By leveraging features like ZFS-based replication, stretched clusters, and advanced snapshot management, you can create a robust infrastructure that meets stringent RPO and RTO requirements. Combine replication with regular backups and proactive monitoring to build a comprehensive disaster recovery strategy tailored to your needs.
