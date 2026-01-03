---
layout: page
permalink: /blogs/os/rocev2.html
title: RoCEv2
---

# RoCEv2: Past, Present and Beyond

InfiniBand (IB), which is the the original RDMA technology, has a custom stack (L1-L7), making it totally incompatible with commodity ready Ethernet/IP infrastructures we have.

RDMA over converged Ethernet (RoCE), on the other hand, is compatible with Ethernet/IP, and is thus more **scalable** and much **cheaper** than IB equipments, making it a good choice for data centers.

Historically, IB is mostly used in the HPC community. More recently, RoCEv2 has been widely utilized in HPC, AI/ML, and storage in data centers.

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

## 2016: TIMELY (Google)

Original Paper: [TIMELY: RTT-based Congestion Control for the Datacenter](https://dl.acm.org/doi/10.1145/2829988.2787510)

Why RTT over ECN?

- RTT directly reflects latency
  - Further, an ECN mark describes behavior at a single switch
- RTT can be measured accurately in practice (using latest NIC hardware features)
- RTT is a rapid, multi-bit signal (capturing the full extent of end to end queueing delay)

TIMELY framework (implemented in host software with NIC hardware support):

1. RTT measurement to monitor the network for congestion
2. a computation engine that converts RTT signals into target sending rates
3. a control engine that inserts delays between segments to achieve the target rate

  ![TIMELY Overview](/blogs/images/os/timely_overview.jpg)

Important assumptions/Why RTT works here?

- Typically, packets going through different network paths have disparate delays, but in datacenters, all paths have small propagation delays
- The turnaround time at the receiver to generate the ACK is almost 0 here, because the NIC hardware instead of system software handles it

  ![TIMELY Result](/blogs/images/os/timely_result.jpg)

Following TIMELY, the author of DCQCN published a [paper](https://link.zhihu.com/?target=https%3A//www.microsoft.com/en-us/research/wp-content/uploads/2016/09/ecndelay-conext16.pdf), arguing ECN-based approach is better than RTT-based approach for RDMA.

## 2020: Swift (Google)

Original Paper: [Swift: Delay is Simple and Effective for Congestion Control in the Datacenter](https://dl.acm.org/doi/10.1145/3387514.3406591)

Swift still adopts a RTT-based approach, adapted and evolved from TIMELY. The authors made three key points

- Delay as a congestion signal has proven effective (with the virtue of simplicity)
- It is important to respond to both fabric and host congestion.
  - TIMELY only considers fabric congestion
  - **Maybe** it's because host congestion is no longer negligible in data center networks which is several µs
- We must support a wide range of traffic patterns including large-scale incast
  - What DCQCN lacks
  - By supporting even a congestion window of less than one

Swift uses many TCP congestion control concepts and ideas (AIMD)

  ![Swift Congestion Control Algorithm](/blogs/images/os/switt_algorithm.jpg)

Key points:

- Compared with TCP (Reno), Swift congestion control acts on every ACK, while TCP acts on congestion (Timeout/3 ACKs)
- On timeout, where TCP would do slow start deterministically, Swift selectively so slow start/fast recovery based on a threshold
- Swift indeed follows the same AIMD philosophy but has much more tunable parameters
- Swift considers both fabric and host congestion, the actual congestion window = `min(fcwnd, hcwnd)`

Fabric Target Delay Scaling:

- Topology-based Scaling:
  - \#Hops matters!
  - Measure the forward path hop count by subtracting the received IP TTL values from known starting TTL, and reflect it back in the ACK header.
- Flow-based Scaling:
  - N flows contending for the same switch
  - Adjusted in proportion to $\frac{1}{\sqrt{cwnd}}$ (Using cwnd to approximate $\frac{1}{N}$)
- Overall target:
  - $\text{base} + f(\text{\#Hops}) + f(\text{cwnd})$

Swift Loss Recovery is also like TCP:

- SACK
- Retransmission Timeouts

## 2018: IRN (Mellanox, UC Berkeley)

The above mentioned congestion control mechanisms naturally assume that a lossless L2 and thus PFC is needed. This paper, [Revisiting Network Support for RDMA](https://dl.acm.org/doi/10.1145/3230543.3230557), tries to prove that the need for PFC is **an artifact** of current RoCE NIC designs **rather than a fundamental requirement**.

improved RoCE NIC (IRN) Design:

- Goal: make minimal changes to the RoCE NIC design in order to eliminate its PFC requirement
- Improving the **loss recovery mechanism**: TCP like SACK
- Basic end-to-end flow control (termed BDP-FC)
  - BDP = bandwidth-delay product of the network
  - Dividing the BDP of the longest path in the network (in bytes) with the packet MTU set by the RDMA queue-pair (typically 1KB in RoCE NICs).

Key comparison with iWARP

- **No** TCP congestion window control, AIMD (although the authors later mentioned that AIMD logic further improves its performance)
- Operates directly on RDMA segments **instead of** bytestream

Implementation:

- RoCE NICs already support per-packet ACKs for Writes and Sends, but not read
  - Add Read ACK/NACK
- Supporting Out-of-order Packet Delivery
  - DMAs OOO packets directly to the final address in the application memory and keeps track of them using bitmaps; \# sized at BDP cap, but still too large for NIC SRAM
  - **First packet issues**: critical information is only carried in the first packet of a message
  - **WQE matching issues**: how to map packets to WQE
  - **Last packet issues**: Finish processing the last packet no longer marks finishing processing the entire message (e.g. before generating CQE)
  - **Application-level Issues**: New writes can be overwritten by retransmission of old writes

## Modern RDMA

### Revisiting RoCEv2 Design

In the past decades, researchers have been revisiting some of the fundemantal design principles in RoCE (mostly on lossless/lossy L2, in-order/out-of-order).

Original Paper: [Datacenter Ethernet and RDMA: Issues at Hyperscale](https://dl.acm.org/doi/10.1109/MC.2023.3261184)

> RoCE uses InfiniBand’s simple transport layer that heavily builds on **in-order delivery** as well as **go-back-n retransmission semantics** that essentially require a highly reliable in-order fabric for efficient operation.

> RoCE's semantics, load balancing, and congestion control mechanisms are inherited from InfiniBand. This implies that all messages should appear at the destination in order **as if they were transmitted over a static route**, essentially disallowing many packet-level load balancing mechanisms.

> For AI training workloads which are long-lived flows, multi-pathing mechanisms can greatly improve the job completion time.

### 2020: Scalable reliable datagram (SRD) (Amazon)

Original Paper: [A Cloud-Optimized Transport Protocol for Elastic and Scalable HPC](https://ieeexplore.ieee.org/document/9167399)

Supplementing Document: [Add SRD documentation](https://github.com/amzn/amzn-drivers/blob/master/kernel/linux/efa/SRD.txt)

Key Design:

- Multipath Load Balancing:
  - Packet spray instead of ECMP (hash collision hotspots)
- Out of Order Delivery:
  - Directly deliver packets to the host
  - Done by messaging layer above SRD
- Congestion control:
  - Prevent queue buildup and packets drops (rather than relying on them for congestion detection)
  - Similar to BBR, based on RTT-indicated per-connection dynamic rate limit

  ![The ideal and maximum FCT for different transfer sizes](/blogs/images/os/amazon_srd_result.jpg)

### 2025+

- Falcon (Google)
  - Original Paper: [Falcon: A Reliable, Low Latency Hardware Transport](https://dl.acm.org/doi/10.1145/3718958.3754353)
- veRoCE (ByteDance)
  - Spec: [字节跳动 veRoCE 传输协议](https://developer.volcengine.com/resource/7584346532149723178)

Both of them:

- Support out-of-order transmission
- Switch from lossless to lossy RDMA
- Support RTT based congestion control

## References

1. [Congestion Control for Large-Scale RDMA Deployments](https://dl.acm.org/doi/10.1145/2829988.2787484)
2. [RDMA over Commodity Ethernet at Scale](https://dl.acm.org/doi/10.1145/2934872.2934908)
3. [TIMELY: RTT-based Congestion Control for the Datacenter](https://dl.acm.org/doi/10.1145/2829988.2787510)
4. [Swift: Delay is Simple and Effective for Congestion Control in the Datacenter](https://dl.acm.org/doi/10.1145/3387514.3406591)
5. [Revisiting Network Support for RDMA](https://dl.acm.org/doi/10.1145/3230543.3230557)
6. [Datacenter Ethernet and RDMA: Issues at Hyperscale](https://dl.acm.org/doi/10.1109/MC.2023.3261184)
7. [Falcon: A Reliable, Low Latency Hardware Transport](https://dl.acm.org/doi/10.1145/3718958.3754353)