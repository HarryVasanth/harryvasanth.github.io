---
layout: post
title: "Proxmox - Virtualisation: Advanced GPU Passthrough with IOMMU Isolation"
date: 2023-09-15 22:14:52 +01
categories: proxmox virtualisation
tags: proxmox gpu iommu pcie virtualisation linux troubleshooting
---

## The Holy Grail of Virtualisation: Near-Native GPU Performance

GPU Passthrough allows a Virtual Machine (Windows for gaming, or Linux for AI/CUDA) to have direct, exclusive access to a physical PCIe graphics card. In a professional setting, this is essential for VDI (Virtual Desktop Infrastructure), hardware-accelerated video encoding, or local LLM (Large Language Model) training.

However, the path to a working passthrough is littered with "Code 43" errors and system hangs. A experienced administrator knows that success depends on two things: **IOMMU Isolation** and **Vendor-Hidden Virtualisation**.

## Phase 1: Enabling the IOMMU Foundation

IOMMU (Intel VT-d or AMD-Vi) is the hardware feature that allows the kernel to isolate a PCIe device from the rest of the system's memory.

### 1. Kernel Parameters

Edit `/etc/default/grub` and add the isolation flags:

```text
# For Intel
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
# For AMD
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

_Note: `iommu=pt` (Pass Through) prevents the kernel from trying to manage every device, which improves performance and stability._

### 2. Loading the VFIO Modules

You must tell Proxmox to use the "Virtual Function I/O" drivers for your GPU instead of the standard NVIDIA/AMD drivers.

```bash
# /etc/modules
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

## Phase 2: Resolving the IOMMU Group Trap

A PCIe device cannot be passed through alone if it shares an **IOMMU Group** with other devices (like a USB controller or a SATA port). If you pass the GPU, you must pass _everything_ in that group, or the system will crash.

**The Diagnostic**:

```bash
# Check groups
for d in /sys/kernel/iommu_groups/*/devices/*; do n=${d#*/iommu_groups/*}; n=${n%%/*}; printf 'Group %s ' "$n"; lspci -nns "${d##*/}"; done
```

If your GPU is in a "dirty" group with critical system components, you have two choices:

1.  **Move the card**: Try a different physical PCIe slot (often the primary slot has better isolation).
2.  **The 'ACS Override' Patch**: A risky kernel patch that forces the kernel to ignore hardware grouping.

```text
pcie_acs_override=downstream,multifunction
```

## Phase 3: Bypassing 'Code 43' (Vendor Workarounds)

NVIDIA and AMD drivers often check if they are running inside a VM. If they detect a hypervisor, they may disable themselves.

**The Advanced VM Config**:
In the Proxmox `.conf` file for your VM (`/etc/pve/qemu-server/XXX.conf`), you must hide the "KVM" signature and spoof the PCIe headers.

```text
cpu: host,hidden=1,flags=+pdpe1gb
hostpci0: 0000:01:00,x-vga=1,pcie=1
args: -cpu 'host,hv_ips,hv_relaxed,hv_reset,hv_runtime,hv_spinlocks=0x1fff,hv_stimer,hv_synic,hv_vapic,hv_vpindex,kvm=off'
```

## Strategy: Handling Audio over HDMI

A GPU passthrough usually includes the HDMI Audio controller. You must pass both the Video and Audio IDs together.

```bash
# Blacklist the default drivers to prevent the host from grabbing the card
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "options vfio-pci ids=10de:1b81,10de:10f0" >> /etc/modprobe.d/vfio.conf
```

## Summary: Testing for Success

1.  **The 'Black Screen' Fix**: Ensure you have disabled the "VGA" output in the VM hardware settings. Once passthrough is active, the GPU _is_ the only output.
2.  **Power Management**: Ensure the GPU is getting enough power. Some high-end cards fail to initialize if the PCIe slot doesn't provide enough "negotiated" wattage before the guest driver takes over.

GPU Passthrough transforms a Proxmox host into a high-performance workstation. By mastering the nuances of IOMMU grouping and hypervisor masking, you can achieve 98-99% of bare-metal performance inside a virtual environment. This is the hallmark of a optimal virtualisation strategy.
