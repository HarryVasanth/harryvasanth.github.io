---
layout: post
title: "Networking - SD-WAN: Core Principles of Path Selection and Multi-Homing in the Enterprise"
date: 2020-01-28 10:10:27 +01
categories: networking architecture
tags: sd-wan networking architecture enterprise routing redundancy
---

## The Transition from Static MPLS to Agile Fabrics

For decades, the standard for enterprise WAN (Wide Area Network) architecture was the high-cost MPLS circuit. While reliable, MPLS acted as a "black box" with virtually no flexibility. If the circuit was saturated by a background backup job, your critical VoIP or video conferencing traffic suffered. If you wanted to add a secondary business broadband link for redundancy, the management of complex BGP or IP SLA configurations often became an operational burden.

**SD-WAN (Software-Defined Wide Area Network)** represents the mandatory transition from hardware-centric routing to application-aware, software-orchestrated networking. This post explores the core architectural principles that define a modern SD-WAN fabric and how they impact global infrastructure management.

## Core Principle 1: Transport Independence (The Overlay)

In a professional network design, we must decouple the logical network from the physical connection. SD-WAN views the underlying link (MPLS, Direct Internet Access, LTE, or even Starlink) as a simple "underlay."

The SD-WAN software creates a secure, virtual "overlay" (typically using IPsec or VXLAN) that spans all available physical links. This allows the network to treat a £500/month business broadband connection with the same logical weight as a £5000/month managed MPLS circuit. Traffic is no longer tied to a specific physical path; it is tied to an application policy. This provides the agility to move workloads between providers without re-architecting the entire subnet structure.

## Core Principle 2: Dynamic Path Selection (DPS)

This is the "intelligence" of the SD-WAN edge. Unlike traditional routing protocols which primarily look at the "Up/Down" state of an interface or a simple cost metric, SD-WAN continuously probes every available path for three critical performance metrics:

1.  **Latency**: The time taken for a round-trip packet.
2.  **Jitter**: The variation in latency, which is the primary killer of real-time application quality.
3.  **Packet Loss**: The percentage of frames that fail to arrive at the destination.

### Implementation: Application-Aware Steering

In the central controller, you define "SLA Classes" for your specific workloads:

- **VoIP/Voice Class**: Latency < 150ms, Jitter < 30ms, Loss < 1%.
- **ERP/Database Class**: Latency < 100ms, Loss < 0.1%.
- **Bulk Data/Backup Class**: No specific requirements.

If the primary link starts experiencing a minor brownout (e.g., jitter rises to 50ms), the SD-WAN edge device identifies the violation within milliseconds and moves the VoIP traffic to a healthier secondary link without dropping the active session. The end-user experiences zero interruption because the session state is maintained across the overlay fabric.

## Core Principle 3: Centralised Orchestration and ZTP

In a traditional environment, changing a Quality of Service (QoS) policy across 100 branch offices requires logging into 100 different routers. This is prone to human error and inconsistent application.

In an SD-WAN fabric, you define your business policy once in a central **Controller** (e.g., Cisco vManage, FortiManager, or VeloCloud Orchestrator). The controller then pushes the configuration to every edge device (CPE) in the fabric. This also enables **Zero-Touch Provisioning (ZTP)**: you can ship a factory-fresh router to a branch office, and as soon as it connects to the internet, it pulls its entire security and routing policy from the cloud. This reduces the need for onsite technical staff during a rollout.

## Advanced Strategy: Local Breakout (Direct Internet Access)

One of the most significant cost-savers in SD-WAN is **Local Breakout**. In legacy hub-and-spoke networks, all branch office internet traffic is "backhauled" through the expensive datacenter MPLS just to reach public cloud services like Office 365 or Salesforce.

The SD-WAN edge device identifies this SaaS traffic and sends it directly out of the local broadband link, reserving the expensive private circuits for internal application traffic.

### The Security Implication

Direct internet access at the branch means your branch office is now an entry point for threats. A professional SD-WAN deployment **must** integrate a **Stateful Firewall**, Intrusion Prevention (IPS), and Web Filtering directly on the edge device to maintain the security perimeter. This is often referred to as SASE (Secure Access Service Edge) when combined with cloud security.

## Performance Optimisation: FEC and Packet Duplication

For critical links with high loss (like satellite or cellular), SD-WAN can perform **Forward Error Correction (FEC)** or **Packet Duplication**.

- **Packet Duplication**: The router sends the exact same packet over two different physical links (e.g., LTE and Broadband) simultaneously. The receiver takes whichever one arrives first and discards the second. This provides 0% loss even on highly unstable connections, at the cost of doubled bandwidth usage.

## Summary

SD-WAN is about moving the intelligence of the network from the core to the edge. It allows you to build a resilient, high-performance WAN using a mix of diverse, commodity internet links. By leveraging dynamic path selection and centralised orchestration, you reduce operational "Toil" while providing a superior experience for the end-user. It is the definitive evolution for any distributed enterprise infrastructure that requires high availability and cloud agility. In 2020, it is the standard architecture for the modern, connected enterprise.
