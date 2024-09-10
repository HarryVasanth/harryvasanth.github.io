---
layout: post
title: "Proxmox - Configure GPU Passthrough for VMs"
date: 2024-09-10 16:19:22 +01
categories: proxmox
tags: proxmox-ve gpu
---

## Intro

GPU passthrough allows virtual machines (VMs) to directly access a physical GPU, significantly enhancing performance for tasks such as gaming, AI workloads, video editing, or hardware-accelerated streaming. This guide provides a detailed and up-to-date walkthrough for configuring GPU passthrough on Proxmox VE, covering AMD, Intel, and NVIDIA GPUs.

---

## Prerequisites and Hardware Compatibility

### **Basic Requirements**

1. **Proxmox VE Version**: Ensure you are using Proxmox VE 8.x or later.
2. **CPU Support**:
   - Intel: VT-d (IOMMU) support.
   - AMD: AMD-Vi (IOMMU) support.
3. **Motherboard**: BIOS must support IOMMU and allow enabling features like "Above 4G Decoding" and "Resizable BAR" (optional for performance boosts).
4. **GPU**:
   - AMD GPUs: Generally more straightforward for passthrough.
   - NVIDIA GPUs: May require workarounds for consumer cards due to driver restrictions (e.g., Error 43).
   - Intel GPUs: Integrated GPUs can also be passed through but may have limited use cases.

### **BIOS Configuration**

1. Enable **IOMMU**:
   - Intel: VT-d.
   - AMD: AMD-Vi.
2. Disable **CSM (Compatibility Support Module)** for UEFI boot.
3. Enable **Above 4G Decoding** and **Resizable BAR** (optional but recommended for modern GPUs).
4. Update your BIOS to the latest version.

---

## Step 1: Enable IOMMU on the Host Machine

### Edit GRUB Configuration

Open the GRUB configuration file:

```bash
sudo nano /etc/default/grub
```

Add or modify the following line based on your CPU:

- **Intel**:
  ```bash
  GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
  ```
- **AMD**:
  ```bash
  GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
  ```

Update GRUB and reboot:

```bash
sudo update-grub
sudo reboot
```

### Verify IOMMU is Enabled

Run the following command to check:

```bash
dmesg | grep IOMMU
```

You should see messages indicating that IOMMU is enabled.

---

## Step 2: Configure PCI Device Isolation

### Identify GPU and Audio Devices

List PCI devices using:

```bash
lspci -nnv | grep -i vga -A 1
```

Note the IDs of your GPU and its associated audio device (e.g., `0000:01:00.0` for GPU and `0000:01:00.1` for audio).

### Blacklist Host Drivers

Blacklist drivers to prevent Proxmox from using the GPU:

Edit `/etc/modprobe.d/blacklist.conf`:

```bash
blacklist nouveau
blacklist nvidiafb
blacklist radeon
```

### Bind Devices to VFIO Driver

Create or edit `/etc/modprobe.d/vfio.conf`:

```bash
options vfio-pci ids=0000:<GPU_ID>,0000:<AUDIO_ID>
```

Update initramfs and reboot:

```bash
sudo update-initramfs -u -k all
sudo reboot
```

### Verify Device Binding

Run the following command to confirm the GPU is bound to `vfio-pci`:

```bash
lspci -nnk | grep -A 3 'VGA'
```

---

## Step 3: Install Necessary Drivers

### NVIDIA Drivers (Host)

For NVIDIA GPUs, install drivers on the Proxmox host:

1. Enable non-free repositories:

   ```bash
   sudo nano /etc/apt/sources.list.d/pve-enterprise.list
   ```

   Add `non-free` at the end of each line.

2. Update packages and install drivers:

   ```bash
   sudo apt update && sudo apt install nvidia-driver nvidia-dkms nvidia-headless-no-dkms
   ```

3. Reboot.

---

## Step 4: Assign GPU to VM in Proxmox UI

1. Open the Proxmox Web Interface.
2. Select the VM you want to configure.
3. Go to **Hardware > Add > PCI Device**.
4. Select your GPU from the list and enable:
   - **All Functions** (binds both GPU and audio).
   - **Primary GPU** if this will be the main display adapter.
5. Save changes and start the VM.

---

## Advanced Features

### **Above 4G Decoding and Resizable BAR**

Enabling these features in BIOS can improve performance, especially for modern GPUs.

### **vGPU or Mediated Device Passthrough**

For NVIDIA enterprise GPUs (e.g., Tesla series) or Intel integrated GPUs with SR-IOV support, you can split a single GPU across multiple VMs using mediated device passthrough.

### **Error 43 Fix for NVIDIA Consumer GPUs**

To bypass Error 43 in Windows VMs:

1. Add this argument in your VM configuration file (`/etc/pve/qemu-server/<VMID>.conf`):
   ```conf
   args: -cpu host,kvm=off,vendor_id=FakeVendorID
   ```
2. Alternatively, use patched drivers or scripts like vGPU unlockers (legal gray area).

---

## Debugging Tips

1. Check IOMMU groups:
   ```bash
   find /sys/kernel/iommu_groups/ -type l
   ```
2. Use `dmesg` logs for troubleshooting driver issues.
3. For AMD GPUs, ensure no reset bugs occur by updating firmware.

---

## Full Automation with PECU Script

To simplify setup, use the [Proxmox-Enhanced-Configuration-Utility (PECU)](https://github.com/Danilop95/Proxmox-Enhanced-Configuration-Utility). This script automates driver installation, IOMMU configuration, and VM setup.

---

## Conclusion

GPU passthrough on Proxmox VE unlocks powerful virtualization capabilities but requires careful configuration tailored to your hardware. With this comprehensive guide, you should be able to set up passthrough successfully for AMD, Intel, or NVIDIA GPUs while leveraging advanced features like SR-IOV or vGPU where applicable.

Test thoroughly before production use!
