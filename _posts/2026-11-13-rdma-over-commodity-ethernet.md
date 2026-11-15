---
title: "Paper Notes: RDMA over Commodity Ethernet at Scale (SIGCOMM'16)"
topic: Paper Notes
description: >-
  A long-form reading of Microsoft's SIGCOMM'16 paper on running RoCEv2 across its data centers.
  It covers the background the paper assumes (why TCP fell short at 40 GbE, why RDMA needs a
  lossless network, how PFC and its headroom work, DCQCN), then the paper: DSCP-based PFC, four
  failures of lossless operation (a go-back-0 livelock, a flooding-induced PFC deadlock, NIC pause
  frame storms and a slow-receiver symptom), the monitoring and rollout behind it, and its results
  (99th percentile latency of 90 µs against 700 µs for TCP; 3 Tb/s with no drops). It closes with
  what came next (PFC watchdogs, Azure's region-wide RDMA, Meta's PFC-only AI fabric, and lossy
  transports such as IRN, SRD, Falcon and Ultra Ethernet) and on what a lossless network costs.
---

# Paper Notes: RDMA over Commodity Ethernet at Scale (SIGCOMM'16)

My [paper notes](/blogs/osdi-2026-notes.html) usually give a paper a few hundred words. This one gets a post of its own: it is Microsoft's account of running RDMA across a routed Ethernet data center network, the first public report of RoCEv2 at that scale, and the problems it describes still shape how RDMA networks are built ten years later.

*Chuanxiong Guo, Haitao Wu, Zhong Deng, Gaurav Soni, Jianxi Ye, Jitendra Padhye and Marina Lipshteyn, all at Microsoft (industry)* · SIGCOMM'16 · [Paper](https://dl.acm.org/doi/10.1145/2934872.2934908) · [PDF](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/rdma_sigcomm2016.pdf)
{: .paper-meta}

- RDMA
- SIGCOMM
{: .paper-tags}

**In one sentence.** Microsoft made RoCEv2 routable by carrying the PFC priority in the IP header (DSCP) instead of a VLAN tag, then found and contained the ways a lossless network fails in practice (a transport livelock, a PFC deadlock, NIC pause frame storms and slow receivers) with NIC and switch fixes, watchdogs and monitoring.

**How to read it.**

- Section 1 is background; sections 2 to 6 follow the paper; section 7 is what came after; section 8 has my final remarks.
- Numbers are the paper's unless marked as my estimates. My comments are under **My take**.

## Contents
{: .no_toc}

* TOC
{:toc}

## 1. Background

### 1.1 Existing TCP optimizations are not enough

> **Takeaway:** Various NIC offload techniques cut TCP's per-packet costs, but at 40 GbE TCP still burned 2 to 4 cores and paid in tail latency for occasional drops. RDMA takes the kernel off the data path entirely.

The usual offloads, all moving work into the NIC:

- **Checksum offload.** The NIC computes and verifies the TCP and IPv4 header checksums, so the kernel never reads the payload just to checksum it. (The Ethernet FCS was always the NIC's job.)

    ![A TCP/IPv4 packet: Ethernet header, IPv4 header with its checksum field, TCP header with its checksum field, payload, and the Ethernet FCS trailer. The IPv4 header checksum and the TCP checksum, which also covers the payload and a pseudo-header, are marked offloaded; the CRC-32 FCS over the whole frame is marked always NIC](/assets/images/posts/rdma-over-commodity-ethernet/checksums.svg)

- **Large segment offload (LSO, or TSO in Linux).** The kernel hands the NIC up to 64 KB with one header, and the NIC cuts it into MSS-sized packets: about 45 times less per-packet work in the kernel. GRO and LRO do the reverse on receive.
- **Receive side scaling (RSS).** The NIC hashes each packet's 5-tuple, looks it up in an indirection table, and steers it to a per-core receive queue. Receive work spreads across cores, while each flow stays on one core (in order, cache-local). It does not speed up a single flow.
- **Interrupt moderation.** One interrupt per batch of packets, or per short timer, instead of one per packet.

At 40 GbE, these were not enough:

- **CPU.** Sending at 40 Gb/s over 8 TCP connections took 6% of a 32-core server's CPU, and receiving took 12%: about 2 and 4 cores just to move bytes, mostly on copies and per-segment protocol work.
- **Latency.** The kernel stack can add up to tens of milliseconds. Bursty traffic also causes occasional drops, and their recovery lands in the tail.
- **RDMA removes both.** The NIC runs the transport and places data directly in application memory. On a production service, that meant a 99th percentile latency of 90 µs against TCP's 700 µs (section 5.1).

### 1.2 Why RDMA needs a lossless network

> **Takeaway:** RDMA's transport lives in the NIC and was built for a network that almost never drops packets. On Ethernet, something has to make that true.

- For a reliable connection (RC), the NIC itself splits messages into MTU-sized packets numbered by packet sequence number (PSN), and handles ACKs, NAKs and retransmission.
- It keeps that state for thousands of QPs in little on-chip memory, so the InfiniBand transport it implements assumes a fabric that practically never drops:
  - **the receiver expects packets in order;**
  - recovery is go-back-N at best, with no selective acknowledgment (section 3.1 shows a NIC that did worse);
  - the retransmission timer is coarse: the Local ACK Timeout is 4.096 µs × 2<sup>n</sup>, so n = 10 gives about 4 ms and n = 14 about 67 ms, hundreds to thousands of data center round trips.

### 1.3 InfiniBand, iWARP and RoCEv2

> **Takeaway:** RoCEv2 keeps InfiniBand's simple NIC transport but runs it over routed UDP/IP: only the NICs are special, and PFC makes the Ethernet underneath lossless.

| | InfiniBand | iWARP | RoCEv2 |
|---|---|---|---|
| Runs over | InfiniBand's own link and network layers | TCP/IP over Ethernet | UDP/IP over Ethernet (port 4791) |
| Specialized hardware | NICs, switches, routers | NICs | NICs |
| Lossless by | credit-based flow control | not needed: TCP handles loss | PFC |
| Retransmission | go-back-N | TCP's own, in the NIC: far more hardware and state | go-back-N |
| Multipath | routes from the subnet manager | ECMP | ECMP, one path per QP |
| Microsoft's verdict | no IB routers in production, and not Ethernet | inherits TCP's latency under loss | routable over the existing network: chosen |

**One path per QP.**

- The UDP source port is fixed per QP, so all of a QP's packets follow one path and stay in order, as go-back-N needs. Different QPs, even between the same hosts, can take different paths.
- The flip side is ECMP collisions, which cap throughput at 60% in section 5.2. Header pictures are in my [RoCEv2 post](/blogs/os/rocev2.html#rocev2-packet).

### 1.4 The network: a three-tier Clos

> **Takeaways:**
> - In 2016, RDMA ran only inside one data center, among servers under the same Spine layer: no PFC hop longer than about 300 m, and round trips of microseconds.
> - The routed three-tier Clos with up-down routing is deadlock-free on paper, and its 300 m Leaf-Spine links make PFC expensive.

![A three-tier Clos network: tens to hundreds of Spine switches on top, Leaf switches in the middle and ToR switches at the bottom, grouped into podsets, with servers under each ToR. Leaf-Spine links are 200 to 300 m of fiber (about 15 KB in flight per round trip at 40 Gb/s), ToR-Leaf links 10 to 20 m (about 1 KB), and server-ToR links about 2 m of copper (about 100 B)](/assets/images/posts/rdma-over-commodity-ethernet/clos.svg)

- **Tiers.** 20 to 40 servers per ToR; tens of ToRs under a set of Leaves (a podset); the Leaves connect to tens to hundreds of Spines. One data center holds tens to hundreds of thousands of servers.
- **Links.** Mostly 40 GbE: about 2 m from server to ToR, 10 to 20 m from ToR to Leaf, 200 to 300 m from Leaf to Spine.
- **Routing.** Every switch routes IP. Layer 2 survives only on the last hop: each ToR's servers form one subnet, reached through the ToR's ARP and MAC tables. That is where section 3.2's deadlock starts.
- **Up-down routing.** A packet climbs to a common ancestor of source and destination, then descends, and never goes down and then up. So buffer dependencies cannot form a cycle, and PFC, in theory, cannot deadlock.
- **Scope.** Traffic between data centers, and legacy applications, stayed on TCP.

### 1.5 PFC: making Ethernet lossless

> **Takeaways:**
> - PFC makes Ethernet lossless one hop at a time with two watermarks: when a priority's ingress buffer crosses **XOFF**, the receiver pauses that priority at the sender; when it falls below **XON**, it lets the sender resume.
> - Every lossless priority on every port must also reserve headroom for the data still arriving after XOFF. Headroom grows with the MTU, the PFC reaction time and, above all, the propagation delay, which is worst on the 300 m Leaf-Spine links. Since it all comes out of one shared switch buffer, 9 to 12 MB **could afford only two lossless classes**.

![An upstream egress port with one queue per priority sends priority-3 data packets over a cable of 2 m to 300 m to a downstream ingress port. The ingress counter for that priority is drawn as a bar with an XON threshold, a higher XOFF threshold, and a headroom region above XOFF. Crossing XOFF sends a PAUSE for priority 3 back upstream, which pauses only that priority's queue; falling below XON sends a PAUSE with time 0. The headroom absorbs the bytes still on the cable and those sent before the upstream reacts](/assets/images/posts/rdma-over-commodity-ethernet/pfc-headroom.svg)

**The mechanism.** Priority-based Flow Control (PFC, IEEE 802.1Qbb):

- **Per link, per priority.** PFC runs between the two ends of one link, separately for each of 8 priorities. These are the 3-bit PCP values of the 802.1Q VLAN tag (DSCP, after section 2). The operator chooses which priorities are lossless.
- **Who marks the priority.** The sender. For RoCEv2, the NIC builds the headers, so it writes the mark itself, from the QP's traffic class or a per-device default. For TCP, the kernel writes it from socket options. Switches classify packets by the mark, and can rewrite it.
- **XOFF and XON.** The ingress port counts the bytes it holds for each priority; in shared-buffer switches this is just a counter over one common pool. Above XOFF, it sends a pause frame for that priority, and the upstream egress queue stops. Below XON, it sends a pause with zero duration, and the queue resumes.
- **The pause frame.** A MAC control frame (EtherType 0x8808) with a priority-enable vector and a timer per priority, counted in quanta of 512 bit times: at most about 0.84 ms at 40 Gb/s, refreshed while congestion lasts.

**Headroom.**

- After XOFF, data keeps arriving: the bytes on the cable, those sent before the pause takes effect, and up to a maximum-size frame at each end. Each lossless priority on each port must reserve buffer for it.
- It is a per-port, per-priority setting on the switch (and on the NIC), usually derived from link speed and cable length.
- My estimate of the cost: a 300 m link has a 3 µs round trip, 15 KB at 40 Gb/s, so about 20 KB with the MTU and reaction time. A switch with 32 such ports needs about 0.64 MB per lossless priority, or about 5 MB for all eight: more than half of a 9 MB buffer, sitting idle.
- **Only two lossless classes.** Microsoft's 9 to 12 MB ToR and Leaf switches therefore ran two: one for real-time RDMA traffic and one for bulk transfer. TCP went into a separate lossy class with reserved bandwidth.

### 1.6 What PFC breaks, and DCQCN

> **Takeaway:** DCQCN is ECN-based congestion control implemented in the RDMA NIC. It keeps queues short so that PFC rarely fires, but PFC remains the last line of defense, and PFC's failure modes are this paper's subject.

- **What PFC breaks.** PFC pauses a whole port and priority, not a flow:
  - under persistent congestion, pauses climb hop by hop back toward the sources (**congestion spreading**);
  - flows that merely share a port with a congested one are paused too (**head-of-line blocking**, **victim flows**).
  - My [RoCEv2 post](/blogs/os/rocev2.html#2015-dcqcn-microsoft-mellanox) has pictures of both.
- **DCQCN** (SIGCOMM'15, with several of the same authors):
  - switches ECN-mark packets when their queues build;
  - the receiving NIC answers with Congestion Notification Packets (CNPs);
  - the sending NIC cuts its rate, and recovers it later.
  - Microsoft chose it because it reacts directly to switch queue lengths, and every switch it used supported ECN well.

## 2. DSCP-based PFC: from VLANs to IP

> **Takeaway:** DSCP-based PFC scales where VLAN-based PFC could not. The Clos routes IP, and a VLAN tag does not survive a router hop while DSCP does; it also removes the VLAN configuration on servers and switches that broke PXE boot.

![Three packet layouts. (a) VLAN-based PFC: the priority is the 3-bit PCP field of the 802.1Q tag, which also holds a 16-bit TPID of 0x8100, a 1-bit DEI and the 12-bit VLAN ID. (b) DSCP-based PFC: no VLAN tag; the priority is the 6-bit DSCP field in the IPv4 header's ToS byte, next to the 2-bit ECN field. The PFC pause frame, the same in both, goes to 01:80:C2:00:00:01 with EtherType 0x8808, opcode 0x0101, a priority-enable vector and eight pause timers, and has no VLAN tag](/assets/images/posts/rdma-over-commodity-ethernet/vlan-vs-dscp.svg)

**The problem.**

- In the original design, the priority is the PCP in the 802.1Q VLAN tag, which also carries the 12-bit VLAN ID; one cannot be sent without the other.
- So PFC needed VLANs configured on servers and switches, with server-facing ports in **trunk mode** (tagged frames) instead of **access mode** (untagged). That caused two problems:
  1. **It broke PXE boot.** PXE (Preboot Execution Environment, pronounced "pixie") lets a server boot an operating system installer or a diagnostic image from a network server instead of a local disk; Microsoft installs, upgrades and repairs servers this way. During PXE boot the NIC has no VLAN configuration and sends untagged frames, which a trunk port does not accept, so the server cannot reach the provisioning service.
  2. **It did not survive routing.** The VLAN tag is part of the Ethernet header, which every router hop strips and rebuilds, so the priority is gone after the first router. The IP header travels end to end.

**The fix.**

- PFC pause frames carry no VLAN tag; in data packets, the tag exists only to carry the priority. So move the priority into the IP header's 6-bit DSCP field.
- Only data packets change. Server ports go back to access mode, PXE boot works, and the priority crosses subnets intact.
- NICs and switches classify and queue by DSCP, and NICs stamp DSCP on what they send. Each port has eight priority groups, each lossless or lossy; when lossless group $i$ crosses XOFF, the port pauses priority $i$. Microsoft simply maps DSCP $i$ to priority $i$.
- All major vendors supported it (Arista, Broadcom, Cisco, Dell, Intel, Juniper, Mellanox and others). The cost is protocols that must stay at layer 2, such as FCoE, which Microsoft did not run.

**My take.**

- A small change with a long life: DSCP classification is now the normal RoCE configuration. Meta's 2024 AI training fabric uses "DSCP-based PFC and a single lossless queue across all network switches and NICs".
- With DSCP, each host picks its own class: a host that marks bulk traffic lossless puts it under PFC. The network has to trust or police that byte, which is one reason for the configuration monitoring of section 4.1.

## 3. Four ways a lossless network failed

- The core of the paper (its Section 4) is four failures caused by PFC and the RDMA transport.
- Deadlock was a known risk of PFC, though this one had a new cause; the other three had not been reported before.

| Failure | Root cause | Fix |
|---|---|---|
| Transport livelock | **go-back-0**: after any loss, the NIC restarted the message from its first packet | go-back-N in the NIC |
| PFC deadlock | switches flooded lossless packets for servers whose MAC table entry had expired | drop such packets instead |
| NIC PFC pause frame storm | a NIC receive-pipeline bug made it pause forever | the NIC fix, plus watchdogs in the NIC and the ToR |
| Slow-receiver symptom | misses in the NIC's address-translation cache | 2 MB pages, and dynamic buffer sharing in switches |

### 3.1 RDMA transport livelock

> **Takeaway:** Go-back-0 retransmits a message from its first packet after any loss, so even 0.4% loss drove goodput to zero. Go-back-N, which resumes from the lost packet, fixed it and is now the standard RoCE recovery.

![After packet 5 of a message is lost and packets 6 to 9 are already in flight, NAK(5) reaches the sender. Go-back-0 resends the whole message, including packets 0 to 4 that were already delivered. Go-back-N resends packets 5 to 15. Selective repeat resends only packet 5 and then continues with 10 to 15, because the receiver kept 6 to 9](/assets/images/posts/rdma-over-commodity-ethernet/go-back.svg)

- **The experiment.** PFC removes congestion drops, but FCS errors and switch bugs still drop packets. In the lab, two servers behind one switch exchanged 4 MB messages with SEND, WRITE and READ as fast as possible, while the switch dropped every packet whose IP ID ended in 0xff: one packet in 256 (0.4%).
- **The result.** The link ran at line rate, and goodput was zero: a livelock.
- **The cause: go-back-0.** On NAK(i), the NIC restarted the message from packet 0. A 4 MB message is about 4,000 packets, so no attempt ever reached the end.
  - The vendor chose go-back-0 because it needs no retransmission state, and RDMA assumes a lossless network.
  - Random loss is barely better (my arithmetic): at the same 1/256 rate, a 4,096-packet message gets through in one attempt with probability $(255/256)^{4096} \approx 1.1 \times 10^{-7}$.
- **The fix: go-back-N.** Resume from the first lost packet.
  - It still wastes about one RTT of data per loss, because the in-order receiver discards whatever arrives after the hole; in my estimate, that is about 85% of line rate at 1/256 loss with a 10 µs RTT. But it is almost as simple as go-back-0, and it cannot livelock.
  - Selective acknowledgment did not fit in the NIC, and with PFC it seemed overkill.

**My take.**

- "Lossless" means "no congestion drops", not "no drops". A transport built on the stronger assumption fails badly: 0.4% loss, 0% goodput.
- Go-back-N is still how RoCE recovers, and loss recovery is still slow on some NICs, as both [Azure](https://www.usenix.org/conference/nsdi23/presentation/bai) and [Meta](https://dl.acm.org/doi/10.1145/3651890.3672233) report.
- Making the NIC good at loss is where IRN and the newer transports went (section 7.2).

### 3.2 PFC deadlock in a Clos

> **Takeaway:**
> - A Clos with up-down routing is supposed to be deadlock-free: every packet goes up, then down.
> - But a ToR's ARP entries (4 hours) outlive its MAC entries (5 minutes), so packets for a dead server find an incomplete ARP entry: the switch knows the MAC, but not the port.
> - The switch then floods them out of every port, uplinks included. Lossless traffic goes down and then back up, a loop of pauses closes, and the deadlock-free assumption breaks.

![Two Leaf switches, La and Lb, above two ToRs, T0 and T1. S1 and S2 hang off T0; S3, S4 and S5 off T1; S2 and S3 are dead. S1 sends purple packets to S3 through T0, La and T1, and T1 floods them out of p4, up to Lb. S4 sends blue packets to S2 through T1, Lb and T0, and T0 floods them out of p2, up to La. Numbered PFC pauses form a loop: 1, T1.p3 pauses La.p1; 2, La.p0 pauses T0.p2 and T0 pauses S1; 3, T0.p3 pauses Lb.p0; 4, Lb.p1 pauses T1.p4 and T1 pauses S4](/assets/images/posts/rdma-over-commodity-ethernet/deadlock.svg)

**The trigger: two tables, two timeouts.** A ToR delivers to its servers through two tables:

- the **ARP table** (IP to MAC), maintained by the switch CPU, with a typical timeout of **4 hours**;
- the **MAC table** (MAC to port), learned in hardware, with a typical timeout of **5 minutes**.
- When a server dies, its MAC entry expires long before its ARP entry. The switch still knows the destination MAC, but not the port, and Ethernet's rule for an unknown destination MAC is to **flood** the packet out of every port, uplinks included.

**The loop** (the paper's Figure 4, redrawn above; all packets lossless):

1. S1 sends to the dead S3 through T0, La and T1, and T1 floods those packets, including up to Lb. Meanwhile, incast toward S5 congests T1.p2, so T1 pauses La.
2. La fills up and pauses T0, and T0 pauses S1.
3. S4 sends to the dead S2 through T1, Lb and T0, and T0 floods those packets, including out of the paused T0.p2. They cannot drain, so T0 pauses Lb.
4. Lb pauses T1, and T1 pauses S4.

- Once closed, the loop holds even after the incast clears: four switches pause each other forever, and restarting servers does not help.
- The cause: a flooded packet that came down from a Leaf can leave through another uplink. That is exactly the down-then-up path that up-down routing rules out.

**The fix.**

- Drop lossless packets whose ARP entry is incomplete, instead of flooding them.
- The alternatives were sending them to the switch CPU, or making the MAC timeout longer than the ARP timeout; both cost CPU or slow down noticing dead servers.
- The general rule: never put broadcast or multicast traffic in a lossless class.

**My take.**

- Up-down routing makes a Clos deadlock-free only if every packet actually follows it. The guarantee covers the routing design, not everything the switches do. Flooding, transient routing loops while routes converge, and rerouting around failures can each send a packet down and then up, and one such path is enough for a cycle.
- In operations, the backstop is the PFC watchdog of the next section: it breaks any loop, whatever its cause, by making a stuck queue lossy.

### 3.3 NIC PFC pause frame storm

> **Takeaway:** One broken NIC can pause an entire lossless network: the pauses spread NIC → ToR → Leaf → Spine, and from there back down to every server. Watchdogs on the NIC and on the ToR make that one port lossy within hundreds of milliseconds.

![A Clos with two Spines, four Leaves, four ToRs and their servers, every switch shaded as paused. Server 0's broken NIC pauses its ToR (1); the ToR pauses its other servers and its Leaves (2); those Leaves pause the Spines (3); the Spines pause the other Leaves (4); the Leaves pause their ToRs (5); the ToRs pause their servers (6)](/assets/images/posts/rdma-over-commodity-ethernet/pfc-storm.svg)

- **What happens.** A NIC that keeps sending pauses backs up its ToR, which pauses its other ports, and the pauses spread through the Leaves and Spines to every server (above, after the paper's Figure 5). No pause frame is forwarded: each hop sends its own pauses as its buffer fills. Microsoft saw this several times in production.
- **The cause.** A bug stopped the NIC's receive pipeline. The receive buffer filled, and the NIC sent pauses forever.
- **The fix.** The vendor fixed the bug. Since the next bug would look the same from the network's side, Microsoft also added two watchdogs:
  - **On the NIC.** A separate micro-controller notices that the receive pipeline has stalled (100 ms by default) while the NIC keeps pausing, and disables pause generation for good: a NIC in this state was never seen to recover. This does not save the server; it protects the network.
  - **On the ToR.** If a server-facing queue cannot drain while the NIC keeps pausing it, the switch makes the port lossy and drops its lossless packets. It restores lossless mode once the pauses have stopped for 200 ms (by default).
  - Either would stop a storm; both together are "double insurance". Repairing the server takes tens of minutes, and the watchdogs limit the damage to hundreds of milliseconds.
- **The incident** (the paper's Section 6.2).
  - A customer's service lost availability. The pauses traced to a single failing server whose ToR saw more than 2,000 pause frames per second from it, while it carried no data.
  - Half of the customer's servers were affected.
  - Storms were rare, a single-digit count across hundreds of thousands of servers. There were none after the watchdogs.

**My take.**

- The watchdog is a circuit breaker: when a port can no longer be lossless, it turns lossy, trading a few drops on one port for the rest of the network.
- The NIC gets no second chance, while the switch port does; that asymmetry records experience.
- It is standard now. SONiC ships it as `pfcwd`, other switch vendors have their own versions, and Azure also runs one on FPGA cards between its ToRs and servers.

### 3.4 The slow-receiver symptom

> **Takeaway:** The NIC only caches its address-translation table; once registered memory outgrows the cache, the NIC stalls and pauses the network. Huge pages fix it, as they do for CPU TLBs.

![Inside an RDMA NIC: packets from the ToR enter an RX buffer and a receive pipeline, which needs a virtual-to-physical translation (1). The on-NIC caches hold QP contexts, work requests and an MTT of only 2K entries; on an MTT miss, the NIC fetches the entry over PCIe Gen3 x8 from the full table in host DRAM (2). Meanwhile the pipeline stalls and the RX buffer crosses XOFF (3), so the NIC sends a PAUSE to the ToR (4). MTT reach: 2,048 entries times 4 KB pages is 8 MB; times 2 MB pages, 4 GB](/assets/images/posts/rdma-over-commodity-ethernet/slow-receiver.svg)

- **The surprise.** A 40 GbE NIC on PCIe Gen3 x8 (about 64 Gb/s), with no CPU on the receive path, should never need to pause its ToR. Yet many servers sent **thousands of pause frames per second**.
- **The cause.**
  - The NIC keeps its QP contexts, WQEs and address-translation table (MTT) in host memory, and caches only a few entries on chip. Its MTT cache had **2K entries**, which at 4 KB pages covers 8 MB.
  - On a miss, the NIC fetches the entry over PCIe, the receive pipeline stalls, the receive buffer crosses XOFF, and the NIC pauses its ToR.
  - The paper calls it a "soft" bug: milder than a storm, but its pauses still spread.
- **The fixes.**
  - 2 MB pages on the hosts, so that 2K entries cover 4 GB.
  - Dynamic buffer sharing in the switches, so a ToR absorbs an occasional server pause locally instead of passing it on.

**My take.**

- An RDMA NIC is a cache over host memory: QP contexts, WQEs and translation entries all live in DRAM, with only a few cached on chip. Once the working set outgrows the cache, throughput falls off a cliff. There are two ways out: fit more entries on chip (smaller per-QP contexts, for example), or make each entry cover more memory (huge pages). It is the CPU's TLB-reach problem, on a NIC.
- In a lossless network, any delay inside a host comes out as PFC pauses, and those pauses put pressure on the whole network.

## 4. Operating RDMA

### 4.1 Monitoring and a staged rollout

> **Takeaway:** Monitoring was built together with the design, and RDMA went from a lab to test clusters to production one tier at a time: only the PFC storm and a few other bugs got as far as production.

**Monitoring.**

- **Configuration.**
  - On switches, PFC is part of the QoS configuration: buffer reservations, DSCP to traffic class to queue mappings, and per-port PFC enables.
  - On servers, settings for RoCEv2, PFC and DCQCN, plus which traffic gets PFC protection, selected by destination transport port.
  - A monitoring service checks that every device's running configuration matches the desired one.
- **Counters.** Pause frames sent and received, plus pause intervals on servers (the switches of the time could not report them), and traffic and drops per port and priority. RDMA packets should never be dropped, so any drop counter that moves is an alarm.
- **RDMA Pingmesh.** Like TCP Pingmesh (SIGCOMM'15): servers exchange 512-byte RDMA probes at ToR, podset and data center scope, and log the RTT or the error code.

**The rollout.** No large RoCEv2 deployment existed when the work began, three years before the paper.

| Step | Where RDMA and PFC ran | What it caught |
|---|---|---|
| 1 | a lab network of tens of servers | the transport livelock and most of the bugs |
| 2 | test clusters, set up and managed like production | the PFC deadlock and the slow-receiver symptom |
| 3 | production, RDMA within a ToR | |
| 4 | production, PFC up to the Leaves | |
| 5 | production, PFC up to the Spines | |

- Each production step went through Microsoft's safe deployment procedure, data center by data center.
- The test clusters ran production's management and monitoring, which "turned out to be invaluable": the monitoring itself was tested before RDMA went live.

### 4.2 An incident: a buffer parameter set too small

> **Takeaway:** Two ToRs under heavy incast ran short of buffer because a buffer-sharing parameter, $\alpha$, was too small, and their pauses raised latency across many services. Raising $\alpha$ fixed it.

- **The knob.** A switch's buffer is shared by its ports. $\alpha$ controls how much of the unallocated buffer a port can take: it may keep growing while it holds less than $\alpha$ times the unallocated buffer. The smaller $\alpha$, the sooner a port crosses XOFF and sends pauses.
- **The incident** (the paper's Section 6.2). On 07/12/2015, the latencies of many latency-sensitive services jumped, and many servers received up to 60,000 pause frames in 5 minutes.
- **The cause.**
  - Two ToRs hosted "chatty" servers that queried more than a thousand servers at once, so the responses came back as a huge incast.
  - These ToRs were a new switch model, whose default $\alpha$ was 1/64 instead of the 1/16 Microsoft had assumed. Their congested ports got far less buffer and paused early, and the pauses spread to thousands of servers.
- **The fix.** Set $\alpha$ back to 1/16, so the congested ports of the two ToRs could take more buffer.

**My take.** Switch defaults are configuration to pin, not to inherit, which is what configuration monitoring is for.

## 5. How RDMA performed

### 5.1 Latency in production

> **Takeaway:** On the same production service, RDMA's 99th percentile latency was 90 µs, against 700 µs for TCP.

- **The service.** Highly reliable and latency-sensitive, with 20K servers per data center, and half its traffic on TCP, half on RDMA.
- **The load.** Peak traffic was about 350 Mb/s per server, at 20 to 30% CPU. The network was not a bottleneck, but the traffic was bursty, with incast.

| | 99th percentile | 99.9th percentile |
|---|---|---|
| TCP | 700 µs, with spikes to several ms | not reported |
| RDMA | 90 µs | about 200 µs |

- RDMA's 99.9th percentile was below TCP's 99th. The incast was the same; RDMA removed the kernel overhead and the drops behind TCP's tail.

### 5.2 Throughput across two podsets

> **Takeaway:** 3 Tb/s across two podsets with zero drops and near-zero CPU; ECMP hash collisions, not PFC, capped utilization at 60%.

- **The setup.** Two podsets of a new, still empty data center, all links 40 GbE.
  - Each podset had 4 Leaves and 24 ToRs of 24 servers, with 6:1 oversubscription at the ToRs and 3:2 at the Leaves.
  - Each podset had 64 × 40 Gb/s of uplinks to the Spines.
- **The traffic.** ToR $i$ in one podset was paired with ToR $i$ in the other. In every ToR, 8 servers each opened 8 RDMA connections to a partner server, and all sent at full speed: 3,074 connections over the 128 Leaf-Spine links.
- **The result.**
  - **3.0 Tb/s, 60% of the 5.12 Tb/s** available, **without a single drop**.
  - Every server sent and received at 8 Gb/s, with CPU close to 0%.
  - The 60% is ECMP hash collisions, not PFC: Microsoft's simulations, and Hedera's results on networks without PFC, show the same.

**My take.**

- ECMP pins each QP to one path, so long-lived, line-rate flows that collide stay collided.
- AI training made this worse, with very low entropy in its 5-tuples. Meta hashes on the destination QP number too (Enhanced ECMP) and spreads each collective over more QPs, for up to 40% faster AllReduce; centralized traffic engineering adds another 5 to 10%.
- The lossy transports spray packets across paths instead (section 7.2).

### 5.3 Lossless is not low latency

> **Takeaway:** Lossless prevents drops, not queues: under load, RDMA's 99th percentile still rose from 50 µs to 400 µs.

- **The testbed.** Two ToRs of 24 servers, each with 4 uplinks to four Leaves (6:1 oversubscription).
- **The traffic.** 20 servers per ToR, each paired with one under the other ToR, with 8 RDMA connections per pair, sending and receiving at 7 Gb/s.

| RDMA latency | before the traffic | during the traffic |
|---|---|---|
| 99th percentile | 50 µs | 400 µs |
| 99.9th percentile | 80 µs | 800 µs |

- TCP's 99th percentile stayed at 500 µs throughout, because RDMA and TCP used separate switch queues.
- RDMA is not a panacea: queues still build under congestion, and its latency grows with them.

## 6. The authors' lessons and open problems

> **Takeaway:** Prove designs in practice, expect the NIC to be the hard part, build monitoring first, and question whether lossless is needed at all.

From the paper's Section 6.3:

- **"A design works in theory is not enough."** Deadlock, livelock and pause propagation all happened, and none was foreseen.
- **NICs are the key.** Most bugs were in the NICs:
  - they implement the complex parts, the verbs and the transport, while the switches' parts are simple (PFC) or well tested (ECN);
  - they cache their state from host DRAM, which brings both bugs and bottlenecks.
- **Be prepared for the unexpected.** Monitoring was in production before customers used RDMA, and it found every bug and incident above.
- **Is lossless needed?** Programmable hardware might make better transports and forward error correction affordable.

Open problems, from its Section 8.1:

- RDMA between data centers, since a PFC hop is limited to 300 m.
- Beating ECMP's 60%, with multipath or per-packet routing in a lossless network.
- A systematic study of deadlock, since local rerouting (as in F10) and many other topologies break up-down routing.
- Low latency and high throughput at the same time.

## 7. Ten years later

> **Takeaway:** Both answers run in production today: lossless RoCE where operators own the whole fabric, and lossy, multipath transports where they build their own NICs.

### 7.1 Lossless RoCE kept spreading

**Azure** ([NSDI'23](https://www.usenix.org/conference/nsdi23/presentation/bai)):

- Enabled RDMA for storage backend traffic in 2018, for frontend traffic in 2019, and between a region's data centers in 2020. By February 2023, about **70% of all traffic in Azure's public regions was RDMA**.
- The recipe is still this paper's: PFC plus DCQCN, two lossless priorities, and PFC watchdogs.
- **The 300 m limit fell.** Region-level links run tens of kilometers, with RTTs up to 2 ms. Azure uses chassis switches with deep, off-chip buffers, and one headroom pool shared by all lossless queues, oversubscribed on the bet that not every port pauses at once.
- **NIC heterogeneity became the hard part.** Three generations of NICs, with three DCQCN implementations, needed firmware updates before they behaved well together.

**Meta** ([SIGCOMM'24](https://dl.acm.org/doi/10.1145/3651890.3672233)):

- Built a RoCEv2 backend network for AI training, including a 24,000-GPU cluster on which a large Llama 3 model was trained with 16,000 GPUs. It uses DSCP-based PFC, a single lossless queue and go-back-N.
- **It dropped DCQCN.** At 200G, DCQCN was hard to tune for collectives; at 400G, a firmware change brought DCQCN bugs, and Meta turned it off.
- **It runs on PFC alone.** The collective library posts an RDMA write only after the receiver has granted it a buffer (a clear-to-send), which caps the traffic in flight. In four years, Meta has not seen its spines persistently pause the rack switches.

### 7.2 ...and the lossy side built transports that do not need it

The paper's last question, whether RDMA needs a lossless network at all, became a research and industry program:

- **IRN** ([SIGCOMM'18](https://dl.acm.org/doi/10.1145/3230543.3230557)): selective retransmission, plus a cap of one bandwidth-delay product in flight, makes RoCE work well without PFC ([my notes](/blogs/os/rocev2.html#2018-irn-mellanox-uc-berkeley)).
- **AWS SRD** ([IEEE Micro, 2020](https://ieeexplore.ieee.org/document/9167399)) sprays each flow's packets across many paths, delivers them out of order, and recovers losses itself, on AWS's own NICs.
- **Google Falcon** ([SIGCOMM'25](https://dl.acm.org/doi/10.1145/3718958.3754353)) and **ByteDance's veRoCE** both accept out-of-order delivery and lossy networks ([my notes](/blogs/os/rocev2.html#2025)).
- **Ultra Ethernet** 1.0 (June 2025) specifies a transport with packet spraying, out-of-order delivery and selective retransmission, designed to run without PFC.

## 8. Final remarks

- **A paper that shaped what came after.**
  - It was the first public account of RDMA over Ethernet at data center scale, and it is unusually candid for an industry paper: a livelock, a deadlock, storms and production incidents, each with its root cause and fix.
  - Much of it became standard practice: DSCP-based PFC, go-back-N, PFC watchdogs, RDMA-aware monitoring and staged rollouts. Azure and Meta (section 7) still build on this recipe.
  - Its closing question, whether RDMA needs a lossless network at all, set the agenda for the lossy transports that followed.
- **Lossless proved heavy.**
  - It costs buffer: headroom for every lossless priority on every port, which left room for only two lossless classes.
  - It costs operations: configuration that must match on every switch and NIC, pause and drop counters, and watchdogs.
  - Worst, it turns local faults into global ones. Every failure in this paper starts local (a dead server, a buggy NIC, a cache miss, one wrong default), and PFC's backpressure amplifies it across the network.
  - Every fix contains the damage by giving up losslessness locally: drop instead of flood, turn a port lossy, absorb pauses in a shared buffer. A lossless network stays safe only because it is willing to drop, which is the lossy camp's argument: if the network must drop safely anyway, make the transport good at drops.

## References

1. Chuanxiong Guo et al. [RDMA over Commodity Ethernet at Scale](https://dl.acm.org/doi/10.1145/2934872.2934908). SIGCOMM 2016.
2. Yibo Zhu et al. [Congestion Control for Large-Scale RDMA Deployments](https://dl.acm.org/doi/10.1145/2829988.2787484) (DCQCN). SIGCOMM 2015.
3. Chuanxiong Guo et al. [Pingmesh: A Large-Scale System for Data Center Network Latency Measurement and Analysis](https://dl.acm.org/doi/10.1145/2785956.2787496). SIGCOMM 2015.
4. [PFC Watchdog Design](https://github.com/sonic-net/SONiC/wiki/PFC-Watchdog-Design). SONiC wiki.
5. Wei Bai et al. [Empowering Azure Storage with RDMA](https://www.usenix.org/conference/nsdi23/presentation/bai). NSDI 2023.
6. Adithya Gangidi et al. [RDMA over Ethernet for Distributed AI Training at Meta Scale](https://dl.acm.org/doi/10.1145/3651890.3672233). SIGCOMM 2024.
7. Radhika Mittal et al. [Revisiting Network Support for RDMA](https://dl.acm.org/doi/10.1145/3230543.3230557) (IRN). SIGCOMM 2018.
8. Leah Shalev et al. [A Cloud-Optimized Transport Protocol for Elastic and Scalable HPC](https://ieeexplore.ieee.org/document/9167399) (SRD). IEEE Micro, 2020.
9. [Falcon: A Reliable, Low Latency Hardware Transport](https://dl.acm.org/doi/10.1145/3718958.3754353). SIGCOMM 2025.
10. [Ultra Ethernet Consortium](https://ultraethernet.org/). Ultra Ethernet Specification 1.0, June 2025.
