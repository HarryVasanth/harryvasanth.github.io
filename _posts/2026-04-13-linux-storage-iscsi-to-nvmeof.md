---
layout: post
title: "Linux - Storage: Transitioning from iSCSI to NVMe-oF TCP for High-Performance Block Storage"
date: 2026-04-13 11:32:01 +01
categories: linux storage
tags: linux storage networking administration
---

## The Bottleneck of Legacy Storage Protocols

For over two decades, iSCSI has been the workhorse of block storage in the traditional datacenter. It provided a reliable way to mount remote storage over standard Ethernet. However, as backend storage media evolved from spinning magnetic disks to SATA SSDs, and finally to PCIe NVMe drives, the protocol itself became the primary bottleneck.

The core issue is translation. The iSCSI protocol wraps SCSI commands inside TCP packets. When those packets reach a modern storage server, the kernel must translate the legacy SCSI commands into NVMe commands. Furthermore, SCSI was designed around a single hardware queue. Modern NVMe drives have up to 64,000 parallel queues. Passing multi-queue NVMe traffic through a single-queue SCSI bottleneck results in high CPU overhead and artificially limits the IOPS (Input/Output Operations Per Second) your hardware can deliver.

**NVMe over Fabrics (NVMe-oF) TCP** is the definitive replacement. It extends the native NVMe command set across a standard IP network. There is no translation layer. The parallel queues of the remote NVMe drive map directly to the parallel CPU queues of your client server.

## The Architecture of NVMe-oF TCP

Unlike earlier NVMe-oF iterations that required expensive RDMA-capable network cards (RoCE or iWARP) and lossless lossless switch configurations, NVMe-oF TCP runs on standard Ethernet. This allows a experienced administrator to build a hyper-converged, ultra-fast storage fabric using standard 10Gbps, 25Gbps, or 100Gbps network interfaces.

The architecture consists of two main components:

1.  **The Target**: The storage server that physically houses the NVMe drives and exports them over the network.
2.  **The Initiator (Host)**: The client server (a hypervisor or database node) that connects to the target and mounts the block device.

## Phase 1: Configuring the NVMe Target

On your storage server, ensure you are running a modern kernel (5.15 or newer is highly recommended for production TCP support). We will use `nvmetcli` to interact with the kernel's configuration filesystem.

```bash
# Install the target management utility
sudo apt update && sudo apt install nvmetcli

```

Next, we load the required kernel modules for the NVMe TCP target:

```bash
sudo modprobe nvmet
sudo modprobe nvmet-tcp

```

To configure the target, we must create a subsystem, add a namespace (the physical block device we want to share), and create a port to listen on. Create a JSON configuration file named `nvmet-config.json`:

```json
{
  "hosts": [],
  "ports": [
    {
      "addr": {
        "adrfam": "ipv4",
        "traddr": "10.10.10.50",
        "treq": "not specified",
        "trsvcid": "4420",
        "trtype": "tcp"
      },
      "portid": 1,
      "referrals": [],
      "subsystems": ["nqn.2026-04.com.example:storage-node-01"]
    }
  ],
  "subsystems": [
    {
      "allowed_hosts": [],
      "attr": {
        "allow_any_host": 1,
        "serial": "123456789",
        "version": "1.3"
      },
      "namespaces": [
        {
          "device": {
            "nguid": "uuid:23531b79-5cd8-4228-a5ec-9f5e0842db13",
            "path": "/dev/nvme0n1"
          },
          "enable": 1,
          "nsid": 1
        }
      ],
      "subnqn": "nqn.2026-04.com.example:storage-node-01"
    }
  ]
}
```

Apply the configuration to the kernel:

```bash
sudo nvmetcli restore nvmet-config.json

```

_Security Note_: In this example, `"allow_any_host": 1` is used for simplicity. In a production environment, you must populate the `hosts` array with the specific NVMe Qualified Names (NQN) of your initiator clients to enforce access control.

## Phase 2: Connecting the Initiator (Host)

On the client server that needs to consume the storage, you must install the native NVMe command line tools.

```bash
sudo apt install nvme-cli
sudo modprobe nvme-tcp

```

First, we discover the available subsystems on our target server:

```bash
sudo nvme discover -t tcp -a 10.10.10.50 -s 4420

```

The output will confirm the target NQN (`nqn.2026-04.com.example:storage-node-01`). Now, we establish the connection:

```bash
sudo nvme connect -t tcp -a 10.10.10.50 -s 4420 -n nqn.2026-04.com.example:storage-node-01

```

Once connected, run `lsblk` or `nvme list`. You will see a new block device, typically named `/dev/nvme1n1`. This device can now be formatted with XFS or ext4, or added to an LVM volume group, exactly as if it were a physical drive plugged into the motherboard.

## Phase 3: High Availability and Multipathing (ANA)

In the iSCSI era, administrators relied on `multipathd` (Device Mapper Multipath) to handle link failures. NVMe introduces a native, much faster alternative known as **Asymmetric Namespace Access (ANA)**.

Because the NVMe driver natively understands multiple paths to the same storage controller, you do not need an external user-space daemon to handle failover. When you connect to the same target NQN over two different network interfaces, the kernel's NVMe subsystem automatically aggregates them into a single block device (e.g., `/dev/nvmeXcXYnZ`).

To enable native NVMe multipathing, ensure the following kernel parameter is set:

```bash
echo "options nvme_core multipath=Y" | sudo tee /etc/modprobe.d/50-nvme-multipath.conf
sudo update-initramfs -u

```

## Performance Tuning: MTU and Interrupt Affinity

To achieve the maximum potential of NVMe over TCP, the network foundation must be flawless.

1. **Jumbo Frames (MTU 9000)**: NVMe block sizes are typically 4KB. Standard 1500-byte Ethernet frames force the kernel to fragment every storage block into three network packets. Enabling MTU 9000 on your switches, target NICs, and initiator NICs allows the entire 4KB block to travel in a single frame. This drastically reduces CPU overhead and interrupt frequency.
2. **Poll Queues**: For ultra-low latency applications (like high-frequency trading databases), you can instruct the NVMe TCP driver to poll the network interface constantly rather than waiting for an interrupt.

```bash
sudo nvme connect -t tcp -a 10.10.10.50 -s 4420 -n nqn.2026-04.com.example:storage-node-01 --nr-poll-queues=4

```

## Summary

The transition to NVMe-oF TCP marks the end of complex, proprietary Storage Area Networks (SANs). By leveraging standard Ethernet infrastructure and native kernel NVMe drivers, you can provide remote block storage that performs almost identically to local PCIe flash. For a system administrator managing massive database clusters or high-density hypervisors in 2026, dropping the legacy SCSI translation layer is a mandatory step toward building a high-performance, cost-effective infrastructure.
