---
layout: post
title: "Proxmox - Implementing Live Migration of Virtual Machines"
date: 2019-12-05 11:50:19 +01
categories: proxmox
tags: proxmox-ve migration
---

## Intro

Live migration in Proxmox VE allows you to move running virtual machines (VMs) between nodes in a cluster with minimal downtime. This feature is essential for load balancing, hardware maintenance, and high availability. In this guide, we’ll explore **advanced concepts** in live migration, including prerequisites, shared storage configurations, troubleshooting common issues, and optimizing the migration process for large VMs.

---

## Step 1: Prerequisites for Live Migration

### **1.1 Cluster Setup**

Live migration requires a Proxmox cluster. To create a cluster:

1. On the primary node:
   ```sh
   pvecm create my-cluster
   ```
2. On additional nodes, join the cluster:
   ```sh
   pvecm add <primary-node-ip>
   ```

Verify the cluster status:

```sh
pvecm status
```

### **1.2 Shared Storage**

Shared storage is required so that VM disk images are accessible by all nodes in the cluster. Supported options include:

- **NFS**:
  ```sh
  apt install nfs-common
  mount <nfs-server-ip>:/shared-storage /mnt/nfs
  ```
- **iSCSI** with LVM:
  ```sh
  iscsiadm -m discovery -t sendtargets -p <iscsi-server-ip>
  iscsiadm -m node --login
  pvcreate /dev/sdX
  vgcreate vg_iscsi /dev/sdX
  ```
- **Ceph RBD** (for distributed storage):
  ```sh
  pveceph install
  ceph-deploy new <node-names>
  ```

Add the shared storage to Proxmox via **Datacenter > Storage > Add**.

### **1.3 Resource Availability**

Ensure the target node has sufficient CPU, memory, and storage to accommodate the migrating VM. Proxmox automatically checks these requirements before migration.

---

## Step 2: Initiating a Live Migration

### **2.1 Using the Web Interface**

1. Navigate to the VM you want to migrate.
2. Click on **Migrate**.
3. Select the target node from the list.
4. Click **Migrate** to start the process.

### **2.2 Using the Command Line**

Run the following command to migrate a VM (e.g., VM ID `103`) from one node to another:

```sh
qm migrate 103 <target-node>
```

---

## Step 3: Understanding the Migration Process

### **3.1 Pre-Copy Phase**

Proxmox begins by copying memory pages from the source node to the target node while the VM continues running.

### **3.2 Stop-and-Copy Phase**

Once most memory pages are copied, Proxmox briefly pauses the VM to synchronize remaining memory pages and CPU state.

### **3.3 Resume on Target Node**

The VM resumes operation on the target node with minimal downtime (usually milliseconds).

---

## Step 4: Advanced Configurations

### **4.1 Optimizing Migration for Large VMs**

For VMs with large memory allocations or high disk I/O:

- Use a high-speed network (e.g., 10GbE) for faster data transfer.
- Enable compression during migration:
  ```sh
  qm migrate --with-local-disks --online --compress <vmid> <target-node>
  ```

### **4.2 Handling Local Disks**

If a VM uses local storage instead of shared storage, add `--with-local-disks` during migration:

```sh
qm migrate <vmid> <target-node> --with-local-disks
```

This transfers local disk data over the network.

---

## Step 5: Post-Migration Verification

After migration:

1. Confirm that the VM is running on the target node via the web interface or CLI:
   ```sh
   qm status <vmid>
   ```
2. Check resource usage on both nodes to ensure proper load balancing.
3. Verify application functionality within the VM.

---

## Step 6: Troubleshooting Common Issues

### **6.1 Shared Storage Not Accessible**

Ensure that all nodes can access shared storage:

- Test NFS mounts:
  ```sh
  ls /mnt/nfs
  ```
- Verify iSCSI connections:
  ```sh
  iscsiadm -m session
  ```

### **6.2 CD/DVD Drive Attached**

Live migration fails if a CD/DVD drive is attached to the VM. Detach it before migrating:

```sh
qm set <vmid> -ide2 none
```

### **6.3 CPU Compatibility Issues**

Ensure CPUs on source and target nodes are compatible:

- Enable CPU flags in `/etc/pve/qemu-server/<vmid>.conf`:
  ```conf
  cpu: host
  ```

---

## Step 7: High Availability and Automation

### **7.1 High Availability (HA)**

Enable HA to automatically migrate VMs in case of node failure:

1. Assign HA roles via **Datacenter > HA > Add**.
2. Configure HA policies (e.g., `restart`, `migrate`).

Verify HA status:

```sh
ha-manager status
```

### **7.2 Automate Migrations with Scripting**

Use Proxmox’s API or CLI tools to automate migrations during maintenance windows.

Example script for migrating all VMs from one node to another:

```sh
#!/bin/bash
SOURCE_NODE="node1"
TARGET_NODE="node2"

for VMID in $(qm list | grep running | awk '{print $1}'); do
    echo "Migrating VM $VMID from $SOURCE_NODE to $TARGET_NODE..."
    qm migrate $VMID $TARGET_NODE --online
done
```

---

## Best Practices for Live Migration

1. **Test Before Production**  
   Test live migration on non-critical VMs before deploying it in production environments.

2. **Use Redundant Networks**  
   Configure multiple network interfaces for migration traffic to avoid disruptions.

3. **Monitor Performance**  
   Use tools like `htop` or Proxmox’s built-in monitoring to track resource usage during and after migration.

4. **Plan Maintenance Windows**  
   Schedule migrations during low-traffic periods to minimize user impact.

---

## Conclusion

Live migration in Proxmox VE is a powerful feature that enables seamless movement of VMs between cluster nodes with minimal downtime, ensuring flexibility and high availability in virtualized environments. By following these best practices, you can optimize your live migration process for even large-scale deployments while maintaining performance and reliability.
