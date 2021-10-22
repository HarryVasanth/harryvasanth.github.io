---
layout: post
title: "Proxmox - Automation: Bulletproof VM Templates with Cloud-init and Terraform"
date: 2021-10-22 05:02:48 +01
categories: proxmox devops
tags: proxmox terraform cloud-init automation iac
---

## The Goal: 60-Second Server Provisioning

Manually installing an OS from an ISO is a waste of a well-versed admin's time. We want a workflow where we can run `terraform apply` and have a fully configured, networked VM ready for use in under a minute.

## Step 1: The 'Golden Image' Template

Instead of a raw ISO, we start with a "Cloud Image" (provided by Ubuntu, Debian, or CentOS).

1.  **Download the Image**: `wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img`
2.  **Create a base VM**: Use `qm create` to define the hardware.
3.  **Import the Disk**: `qm importdisk <VM_ID> jammy-server-cloudimg-amd64.img <STORAGE>`
4.  **Add a Cloud-init Drive**: This is the virtual CD-ROM that Proxmox uses to pass configuration data to the guest.
5.  **Convert to Template**: Right-click the VM and select "Convert to Template."

## Step 2: The Terraform Definition

```hcl
resource "proxmox_vm_qemu" "web_node" {
  count       = 3
  name        = "web-prod-${count.index}"
  target_node = "pve1"
  clone       = "ubuntu-2204-template"

  # Resource allocation
  cores   = 2
  memory  = 2048
  agent   = 1  # Enable QEMU Guest Agent

  # Cloud-init logic
  os_type    = "cloud-init"
  ipconfig0  = "ip=192.168.1.${50 + count.index}/24,gw=192.168.1.1"
  sshkeys    = "ssh-rsa AAAAB3Nza..."
}
```

{: file='main.tf'}

## Troubleshooting Key Considerations

- **The 'Guest Agent' Race**: If Terraform reports success but you can't SSH in, the VM might still be running its first-boot scripts (e.g., updating packages). Always install the `qemu-guest-agent` in your template and wait for it to be active before continuing.
- **Duplicate Machine IDs**: If you don't use Cloud-init to reset the machine ID, all cloned VMs will have the same ID, causing chaos with DHCP leases and log aggregators. Ensure `manage_etc_hosts: true` is in your cloud-config.
- **Resizing Disks**: Cloud-init can automatically resize the root partition to match the size you define in Terraform. Ensure you have the `cloud-utils` or `growpart` package installed in the base template.

## Summary

Treating Proxmox like a cloud provider allows you to leverage DevOps best practices in your on-premise datacenter. Combining Cloud-init with Terraform eliminates manual errors and ensures that every server in your fleet is built to the exact same specification.
