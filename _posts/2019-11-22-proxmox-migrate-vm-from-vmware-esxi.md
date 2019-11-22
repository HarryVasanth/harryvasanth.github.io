---
layout: post
title: "Proxmox - Migrating Virtual Machines from VMware ESXi to Proxmox VE"
date: 2019-11-22 14:05:31 +01
categories: proxmox
tags: proxmox-ve vmware esxi migration
---

## Intro

Migrating virtual machines (VMs) from VMware ESXi to Proxmox VE is a critical step for organizations transitioning to an open-source virtualization platform. Proxmox provides multiple methods for seamless VM migration, including its **Import Wizard**, manual disk conversion, and advanced techniques for minimal downtime. This guide explores migrating VMs from VMware ESXi to Proxmox VE to ensure a smooth transition.

---

## Step 1: Preparing the VMware Environment

### **1.1 Enable SSH on ESXi**

SSH access is required to copy VM files from the ESXi datastore:

1. Log in to the vSphere Client.
2. Navigate to **Host > Configure > Services**.
3. Locate **TSM-SSH**, right-click, and select **Start**.

Alternatively, set the SSH service to start automatically:

```sh
esxcli network firewall ruleset set --enabled true --ruleset-id sshServer
```

### **1.2 Identify VM Files**

Find the storage path of the VM:

1. In vSphere Client, go to **Storage > Datastore > Summary**.
2. Note the path of the VM (e.g., `/vmfs/volumes/datastore50/WinServer2022`).

List VM files via SSH:

```sh
ssh root@<ESXi-IP>
ls -al /vmfs/volumes/datastore50/WinServer2022/
```

Identify the `.vmdk` (virtual disk) and `-flat.vmdk` files.

---

## Step 2: Using Proxmox Import Wizard (Proxmox VE 8+)

The Proxmox Import Wizard simplifies migration by integrating VMware APIs directly into the Proxmox interface.

### **2.1 Add ESXi as an Import Source**

1. Navigate to **Datacenter > Storage > Add > ESXi** in the Proxmox web interface.
2. Enter:
   - **IP Address** of the ESXi host.
   - **Username** and **Password** (e.g., `root`).
3. If using a self-signed certificate, check **Skip Certificate Verification**.

### **2.2 Select and Import VMs**

1. Go to **Datacenter > Storage > [Your ESXi Storage]**.
2. Select the VM you want to import.
3. Click **Import**, then configure:

   - Target storage for disks.
   - Network bridge for VM interfaces.
   - Advanced options (e.g., hardware adjustments or excluding disks).

4. Power down the source VM on ESXi before starting the import.

### **2.3 Verify and Boot**

After importing:

1. Boot the VM in Proxmox.
2. Check hardware compatibility and update drivers if necessary.

---

## Step 3: Manual Migration via Disk Conversion

For older Proxmox versions or custom setups, manually migrate VMs by converting VMware disks.

### **3.1 Export VM Disks**

Copy `.vmdk` files from the ESXi datastore using `scp` or WinSCP:

```sh
scp /vmfs/volumes/datastore50/WinServer2022/*.vmdk root@<Proxmox-IP>:/var/lib/vz/images/<VMID>/
```

### **3.2 Convert `.vmdk` to `.qcow2`**

Use `qemu-img` on Proxmox to convert VMware disks:

```sh
qemu-img convert -f vmdk -O qcow2 /var/lib/vz/images/<VMID>/disk.vmdk /var/lib/vz/images/<VMID>/disk.qcow2
```

### **3.3 Create a New VM in Proxmox**

1. In the Proxmox web interface, create a new VM with similar hardware specs as the original.
2. Replace the default disk with your converted `.qcow2` file:

   ```sh
   mv /var/lib/vz/images/<VMID>/disk.qcow2 /var/lib/vz/images/<VMID>/vm-<VMID>-disk-0.qcow2
   ```

3. Update the VM configuration file (`/etc/pve/qemu-server/<VMID>.conf`) to point to the new disk.

---

## Step 4: Advanced Techniques for Large VMs

### **4.1 Use NFS or CIFS for Shared Storage**

Mount shared storage accessible by both ESXi and Proxmox:

1. Export an NFS share from your storage server or NAS.
2. Mount it on both hosts:

   ```sh
   mount -t nfs <storage-server-ip>:/nfs-share /mnt/nfs
   ```

3. Copy disks directly between hosts via shared storage:

   ```sh
   cp /mnt/nfs/disk.vmdk /mnt/nfs/disk.qcow2
   ```

4. Attach disks to a new VM in Proxmox.

---

## Step 5: Post-Migration Optimization

### **5.1 Install VirtIO Drivers**

For Windows VMs, install VirtIO drivers for better performance:

1. Attach a VirtIO ISO to the VM in Proxmox.
2. Boot into Windows and install drivers for:
   - Network adapters.
   - Disk controllers.

VirtIO ISO download link: https://fedorapeople.org/groups/virt/virtio-win/

### **5.2 Verify Network Configuration**

Ensure that virtual NICs are correctly mapped to Proxmox bridges (e.g., `vmbr0`). Update network settings inside the guest OS if necessary.

### **5.3 Test Performance**

Run benchmarks or stress tests on migrated VMs to ensure they perform as expected.

---

## Step 6: Troubleshooting Common Issues

### **6.1 Boot Issues**

If a migrated VM fails to boot:

- Check disk format compatibility (`qcow2`, `raw`, etc.).
- Ensure that boot order is correctly configured in Proxmox.

### **6.2 Network Connectivity Problems**

Verify that virtual NICs are attached to correct bridges in `/etc/network/interfaces`.

Example bridge configuration:

```conf
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge_ports eth0
    bridge_stp off
    bridge_fd 0
```

---

## Best Practices for Migration

1. **Backup Before Migration**  
   Always create backups of VMs before starting migration.

2. **Test in a Lab Environment**  
   Perform migrations on test VMs before moving production workloads.

3. **Monitor Resource Usage**  
   Ensure that your Proxmox host has sufficient CPU, memory, and storage capacity for migrated VMs.

4. **Update Drivers and Tools**  
   Install guest agents like QEMU Guest Agent for better integration with Proxmox features (e.g., live snapshots).

---

## Conclusion

Migrating VMs from VMware ESXi to Proxmox VE can be achieved through multiple methods, ranging from using the new Import Wizard for seamless integration to manual disk conversion for more control over the process. By following these best practices, you can ensure a smooth transition while minimizing downtime and maintaining performance across your virtualized environment.
