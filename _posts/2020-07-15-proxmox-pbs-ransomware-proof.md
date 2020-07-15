---
layout: post
title: "Proxmox - PBS: Ransomware-Proof Backup Architecture and Offsite Synchronisation"
date: 2020-07-15 05:25:45 +01
categories: proxmox backups
tags: pbs backups security data-integrity disaster-recovery proxmox
---

## The Backup Integrity Paradox

In infrastructure administration, the only thing more dangerous than having no backups is having backups that you _believe_ are valid but fail during a crisis. In a large Proxmox environment with hundreds of Virtual Machines, manually testing restoration is impossible to scale. Furthermore, data on physical disks is subject to "bit-rot"-silent corruption where bits flip over time due to hardware degradation.

Proxmox Backup Server (PBS) addresses these risks through a sophisticated verification architecture that provides mathematical certainty regarding the state of your data. This post focuses on building a ransomware-proof architecture using PBS in 2020.

## How PBS Verification Works

Unlike legacy backup tools that merely check if a file metadata exists, PBS is a content-addressable system. It breaks data into small "chunks" and identifies each chunk by its SHA-256 hash.

A **Verification Job** is a scheduled background process that:

1.  Reads every physical chunk stored in a datastore.
2.  Recalculates the SHA-256 hash of that chunk's data on the disk.
3.  Compares the new hash against the original hash stored in the backup manifest.

If the hashes match, you have proof that the data on the disk is identical to what was sent by the client.

## Implementation: Building a Professional Verification Schedule

Verification is I/O intensive. You should never run it manually; it must be part of your automated data lifecycle.

### 1. Configure the Verification Job

In the PBS Web Interface, navigate to **Datacenter -> Verify Jobs -> Add**.

- **Datastore**: Select your primary storage.
- **Schedule**: Run this weekly during your lowest-activity window (e.g., Sunday at 03:00).
- **Ignore Verified**: Set it to "30 days." This tells PBS only to re-read chunks that haven't been verified in the last month.

### 2. Monitoring the Results

Verification failures should trigger an immediate high-priority alert. Ensure you have configured an **SMTP Relay** to send these reports to your infrastructure team.

## Architecting for Ransomware Resilience: The 'Pull' Sync

A backup server on the same network as your production cluster is a target for ransomware. If an attacker gains administrative access to Proxmox, they can use the credentials stored there to delete your backups on the PBS.

To prevent this, a experienced administrator implements a **Disaster Recovery (DR) Sync** strategy using the "Pull" model.

1.  **Production PBS (Site A)**: Accepts daily backups from the cluster.
2.  **Hardened PBS (Site B)**: Located at a different site or in a secure cloud VPC.
3.  **The Logical Gap**: Site B has no inbound access from Site A. Instead, Site B has a **Sync Job** configured to reach out to Site A and "pull" the encrypted chunks.

**Why this is Bulletproof**: Even if an attacker gains root access to Site A and deletes everything, they have no technical way to reach Site B. The offsite backups are physically and logically protected from the production environment.

## Performance Tuning: The Special Metadata Device

Verification reads every byte of your data. If your PBS is backed by spinning hard drives (HDDs), this can take days.

- **The Pro-tip**: If using ZFS for your PBS storage, you **must** use a mirrored pair of enterprise SSDs as a **Special Metadata Device**. This stores the ZFS metadata and the PBS chunk indexes on flash. This simple hardware addition can speed up verification and garbage collection by 1,000% on HDD-based arrays.

## Summary

A backup is only as good as its isolation and verification. By leveraging PBS's automated verification and implementing a pull-based offsite sync, you move from a state of "hoping the backups are good" to "knowing exactly which bits are safe." This level of rigour is the hallmark of a mature, professional infrastructure deployment in 2020.
