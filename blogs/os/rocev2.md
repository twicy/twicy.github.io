---
layout: page
permalink: /blogs/os/rocev2.html
title: RoCEv2
---

# RDMA over Converged Ethernet version 2(RoCEv2)

InfiniBand (IB), which is the the original RDMA technology, has a custom stack (L1-L7), making it totally incompatible with commodity ready Ethernet/IP infrastructures we have.

RDMA over converged Ethernet (RoCE), on the other hand, is compatible with Ethernet/IP, and is thus more **scalable** and much **cheaper** than IB equipments, making it a good choice for data centers.

## RoCEv2 packet

RoCEv2 encapsulates an RDMA transport packet within an _Ethernet/IPv4/UDP_ packet. This makes RoCEv2 compatible with our existing networking infrastructure. The UDP header is needed for ECMP-based multi-path routing.

- The destination UDP port is always set to 4791, while the source UDP port is randomly chosen for each queue pair.
- The intermediate switches use standard five-tuple hashing. Thus, **traffic belonging to the same QP follows the same path**, while traffic on different QPs (even between the same pair of communicating end points) can follow different paths.

    ![RoCE and RoCE v2 Frame Format Differences](/blogs/images/os/RoCE_and_RoCE_v2_Frame_Format_Differences.png)
    Image source: [Nvidia Documentation on RoCEv2](https://docs.nvidia.com/networking/display/winofv55053000/rocev2)

## Priority-based Flow Control (PFC)

To create a lossless L2, RoCEv2 uses PFC to avoid Ethernet switch overflow by forcing upstream entity to pause data transmission when encountering congestion.

- How it works:
  - PFC is a hop-by-hop protocol between two Ethernet nodes.
  - Packets are queued in up to eight queues, each mapping to a **priority**. At the receiving ingress port, packets are buffered in **corresponding** ingress queues.
  - Once the ingress queue length reaches a certain threshold (XOFF), the switch sends out a PFC _PAUSE frame_ to the corresponding upstream egress queue.
  - Once the ingress queue length falls below another threshold (XON), the switch sends a pause with zero duration to resume transmission.

    ![How PFC works](/blogs/images/os/pfc.jpg)

    Image source: [RDMA over Commodity Ethernet at Scale](https://dl.acm.org/doi/10.1145/2934872.2934908)
- Headroom reservation:
  - Need to reserve space for onflight frames before _PAUSE frame_ hits upstream port
  - Size of the headroom is decided by: MTU, PFC reaction time, propagation delay
- Implementation: **DSCP-based PFC** is favored over VLAN-based PFC (by Microsoft), because layer-3 IP forwarding is more scalable than MAC-based layer-2 bridging

What could go wrong with PFC?

- **PFC Deadlock**:
  - Caused by flooding broadcast of switch when serving packets with valid ARP table entry, invalid MAC address table entry
  - Deadlock of switches sending _Pause Frames_ to upstream port
- **NIC PFC _Pause Frame_ Storm**: a single malfunctioning NIC may block the entire network

## Retransmission: Go-Back-N

- Unlike TCP transports, which assume a lossy L2, therefore implementing state machines and SACK to do retransmission; RoCEv2 and IB transport assume lossless L2, therefore, their retransmission scheme is simply Go-Back-N (historically Go-Back-0).
- Go-Back-N means retransmission starts from the first dropped packet and the previous received packets are not retransmitted.

## 2015: DCQCN (Microsoft, Mellanox)

Original Paper: [Congestion Control for Large-Scale RDMA Deployments](https://dl.acm.org/doi/10.1145/2829988.2787484)

Background:

- PFC is coarse grained, with a granularity of port (and priority) and not flow, leading to a few problems:
  - Elephant flow: flows of different utilization rates are equally paused.
  - Victimflow: Cascading _PAUSE frames_ paralyze irrelavant flows

    ![Problems with PFC](/blogs/images/os/dcqcn_pfc_problems.jpg)
    Image source: [Congestion Control for Large-Scale RDMA Deployments](https://dl.acm.org/doi/10.1145/2829988.2787484)

What DCQCN does:

- Congestion control algorithm implemented in Mellanox NICs (and not the switches)
  - **Reaction Point** (flow sender): When receiving _Congestion Notification Packets (CNP)_, reduce current rate and restore later (fast recovery + linear addition)
  - **Notification Point** (flow receiver): When receiving ECN-marked packets, generate CNP packets (how and when determined by the algorithm mathematically)
  - **Congestion Point** (switch): Mark packet with ECN when suffering from congestion

Note that DCQCN has been implemented in Mellanox NICs, and has been deployed in Microsoft’s datacenters.

Also note that: Though DCQCN helps reduce the number of PFC pause frames, it is **PFC** that protects packets from being dropped **as the last defense**.

## References

1. [Congestion Control for Large-Scale RDMA Deployments](https://dl.acm.org/doi/10.1145/2829988.2787484)
2. [RDMA over Commodity Ethernet at Scale](https://dl.acm.org/doi/10.1145/2934872.2934908)
