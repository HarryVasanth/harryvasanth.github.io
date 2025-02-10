---
layout: post
title: "Proxmox - Implement High Availability Clustering"
date: 2018-09-30 13:40:11 +01
categories: proxmox
tags: proxmox-ve high-availability clustering
---

## Intro

High Availability (HA) clustering in Proxmox ensures that virtual machines (VMs) and containers remain operational even if a node in the cluster fails. This guide covers advanced concepts and step-by-step instructions for setting up a Proxmox HA cluster with shared storage, fencing, and testing failover scenarios.

---

## Step 1: Prerequisites for HA Clustering

Before setting up an HA cluster, ensure the following:

1. **Minimum Three Nodes**: A minimum of three nodes is recommended to maintain quorum and avoid split-brain scenarios.
2. **Shared Storage**: Use NFS, iSCSI, or Ceph for shared storage to allow VMs to migrate between nodes seamlessly.
3. **Redundant Network**: Ensure at least two bonded network interfaces for cluster communication and storage traffic.
4. **Fencing Devices**: Fencing is mandatory to isolate failed nodes and prevent data corruption.

### Example Command to Update Nodes:

```sh
apt-get update && apt-get dist-upgrade -y
```

---

## Step 2: Creating a Proxmox Cluster

### **Step 2.1: Create the Cluster**

On the first node, create the cluster:

```sh
pvecm create my-cluster
```

### **Step 2.2: Add Nodes to the Cluster**

On additional nodes, join the cluster:

```sh
pvecm add <IP_of_first_node>
```

Verify the cluster status:

```sh
pvecm status
```

---

## Step 3: Configuring Shared Storage

Shared storage is critical for HA functionality. In this example, we use NFS.

### **Step 3.1: Configure NFS on the Storage Server**

On the NFS server:

```sh
mkdir -p /export/proxmox
chmod 777 /export/proxmox
echo "/export/proxmox *(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a
systemctl restart nfs-server
```

### **Step 3.2: Mount NFS on Proxmox Nodes**

On each Proxmox node:

```sh
mkdir -p /mnt/nfs
echo "<NFS_SERVER_IP>:/export/proxmox /mnt/nfs nfs defaults 0 0" >> /etc/fstab
mount -a
```

Add the NFS storage in the Proxmox web interface under **Datacenter > Storage**.

---

## Step 4: Enabling High Availability

### **Step 4.1: Enable HA for VMs**

1. Navigate to **Datacenter > HA > Resources**.
2. Add a VM or container to HA using the web interface or CLI:
   ```sh
   ha-manager add vm:<VMID>
   ```

### **Step 4.2: Set Resource Priorities**

Set priorities for HA resources to determine failover order:

```sh
ha-manager set vm:<VMID> --priority <PRIORITY>
```

---

## Step 5: Testing Failover

Testing ensures that your HA setup works as expected.

### **Step 5.1: Simulate Node Failure**

Power off one of the nodes hosting an HA-enabled VM:

```sh
poweroff
```

Check if the VM migrates automatically to another node:

```sh
ha-manager status
```

### **Step 5.2: Monitor Quorum**

Ensure quorum is maintained:

```sh
pvecm status | grep Quorum
```

---

## Step 6: Advanced Configuration

### **6.1 Fencing Setup**

Fencing isolates failed nodes to prevent data corruption.

#### Example Fencing Command:

```sh
fence_node <NODE_NAME>
```

### **6.2 Cluster Maintenance**

Before rebooting a node, stop the HA manager:

```sh
systemctl stop pve-ha-lrm pve-ha-crm
```

After reboot, restart services:

```sh
systemctl start pve-ha-lrm pve-ha-crm
```

---

## Step 7: Monitoring and Troubleshooting

Use these commands for monitoring and troubleshooting:

- Check cluster status:
  ```sh
  pvecm status
  ```
- View HA resource status:
  ```sh
  ha-manager status
  ```
- Debug logs:
  ```sh
  journalctl -u pve-ha-lrm -u pve-ha-crm -f
  ```

---

## Conclusion

Proxmox High Availability Clustering ensures minimal downtime for critical workloads by automatically migrating VMs during node failures. Regularly monitor your setup and test failover scenarios to ensure reliability in production environments.
