---
title: "Paper Notes: Revisiting Network Support for RDMA (SIGCOMM'18)"
topic: Paper Notes
description: >-
  A long-form reading of the SIGCOMM'18 paper that asked whether RDMA needs a lossless network,
  and answered no. It covers what the paper assumes (why RoCE inherited go-back-N and PFC, and why
  iWARP's TCP-in-the-NIC lost to it), then the paper: IRN's two changes to a RoCE NIC (selective
  retransmission and BDP-FC, a cap of one bandwidth-delay product in flight), the simulations
  showing that IRN without PFC beats RoCE with PFC (by 2.8 to 3.7 times without congestion control)
  and that PFC makes IRN worse, the extensions that let a NIC accept packets out of order, and
  what it all costs (3 to 10% more NIC state, and at worst 4 to 7% slower end to end). It closes with what
  came after (selective repeat in NVIDIA's NICs, SRNIC, SRD, Falcon and Ultra Ethernet, while
  Azure and Meta kept PFC) and with where the paper's argument is strong and where it stops.
---

# Paper Notes: Revisiting Network Support for RDMA (SIGCOMM'18)

My [previous notes](/blogs/rdma-over-commodity-ethernet.html) ended with the question Microsoft left open in 2016: does RDMA need a lossless network at all? Two years later this paper answered no. It argues that RoCE needs PFC only because of how its NICs handle loss, and it remains the reference point that later lossy RDMA transports are measured against.

*Radhika Mittal, Alexander Shpiner, Aurojit Panda, Eitan Zahavi, Arvind Krishnamurthy, Sylvia Ratnasamy and Scott Shenker; UC Berkeley, ICSI, Mellanox (industry), NYU and University of Washington* · SIGCOMM'18 · [Paper](https://dl.acm.org/doi/10.1145/3230543.3230557) · [Extended version](https://arxiv.org/abs/1806.08159)
{: .paper-meta}

- RDMA
- SIGCOMM
{: .paper-tags}

**In one sentence.** RoCE needs PFC only because its NIC handles loss badly (a receiver that drops every out-of-order packet, and go-back-N); with selective retransmission and a cap of one bandwidth-delay product in flight per connection, RDMA runs better on a lossy network than RoCE does on a lossless one, for 3 to 10% more NIC state.

**How to read it.**

- Section 1 is background; sections 2 to 5 follow the paper; section 6 is what came after; section 7 has my final remarks.
- Numbers come from the [extended version](https://arxiv.org/abs/1806.08159), whose appendix tables give the exact values behind the paper's bar charts. Values marked ≈ are read off a chart. My estimates are marked as such, and my comments are under **My take**.

## Contents
{: .no_toc}

* TOC
{:toc}

## 1. Background

### 1.1 Where the paper starts

> **Takeaway:** By 2018, RoCE with PFC was the standard way to run RDMA in a data center, and research was busy patching PFC's problems. This paper asks whether PFC was needed in the first place.

- **The deployed answer.** RoCEv2 over a network made lossless by Priority Flow Control (PFC), as in Microsoft's [SIGCOMM'16 paper](/blogs/rdma-over-commodity-ethernet.html). How PFC works, and its headroom cost, are in [section 1.5 of those notes](/blogs/rdma-over-commodity-ethernet.html#15-pfc-making-ethernet-lossless).
- **Its problems.** PFC pauses a whole port and priority, not a flow, which brings head-of-line blocking, congestion spreading, deadlocks and pause storms ([section 3 of those notes](/blogs/rdma-over-commodity-ethernet.html#3-four-ways-a-lossless-network-failed)).
- **The patches.**
  - Congestion control, so that PFC rarely fires: DCQCN (ECN-based) and TIMELY (delay-based), both from 2015.
  - Deadlock avoidance (two HotNets'16 papers, one from HKUST and Microsoft, one from Mellanox) and watchdogs.
  - Resilient RoCE (KBNets'17, Mellanox): RoCE with DCQCN and PFC turned off, hoping congestion control alone avoids the drops.
- **The question.** Not how to fix PFC, but: "can the RoCE NIC design be altered so that we no longer need a lossless network fabric?"

### 1.2 Why RoCE needs PFC

> **Takeaway:** RoCE inherited InfiniBand's transport, which was built for a fabric that practically never drops. Its receiver discards every out-of-order packet and its sender goes back N, so without PFC each drop costs about a round trip of packets resent for nothing.

- **InfiniBand's assumption.** InfiniBand's links use credit-based flow control: a sender transmits only into buffer the receiver has advertised, so the fabric does not drop. Its transport, implemented in the NIC, was therefore never designed to recover from loss efficiently.
- **The receiver.** An out-of-order packet is discarded, and the receiver sends a NACK.
- **The sender.** On a NACK, it resends everything after the last acknowledged packet: **go-back-N**.

![After packet 5 of a message is lost and packets 6 to 9 are already in flight, NAK(5) reaches the sender. Go-back-0 resends the whole message, including packets 0 to 4 that were already delivered. Go-back-N resends packets 5 to 15. Selective repeat resends only packet 5 and then continues with 10 to 15, because the receiver kept 6 to 9](/assets/images/posts/rdma-over-commodity-ethernet/go-back.svg)

- **The price.** At loss rates a lossy network actually sees, go-back-N wastes bandwidth on duplicates, and those duplicates add to the congestion that caused the loss. The [previous notes](/blogs/rdma-over-commodity-ethernet.html#31-rdma-transport-livelock) show an even simpler NIC, go-back-0, reaching zero goodput at 0.4% loss.

### 1.3 iWARP: the road not taken

> **Takeaway:** iWARP handled loss in the NIC from the start, by running TCP there. But mapping RDMA onto a TCP byte stream takes several protocol layers, which made iWARP NICs slower and more expensive, and the market chose RoCE and, with it, the lossless network.

![Three protocol stacks side by side, each with the verbs API in host software on top. iWARP puts RDMAP (Read, Write and Send operations), DDP (places segments directly in memory), MPA (frames segments in a byte stream), TCP and IP in the NIC, and runs over any IP network, with loss handled in the NIC; the Chelsio T580-CR costs $760 and does 3.24 Mpps at 2.89 µs. RoCEv2 puts the InfiniBand transport (BTH header with opcode, QP number and PSN; go-back-N; a receiver that drops out-of-order packets), UDP on port 4791 and IP in the NIC, and needs lossless Ethernet with PFC on every link; the Mellanox MCX416A costs $420 and does 14.7 Mpps at 0.94 µs. IRN keeps RoCEv2's layers and adds selective retransmission with a SACK bitmap, BDP-FC (at most one BDP in flight), out-of-order placement in memory, and Read (N)ACKs and a few header fields; it runs over any IP network, with PFC optional and best left off; it was simulated and its logic synthesized on an FPGA, with 3 to 10% more NIC state and at least 45 Mpps](/assets/images/posts/revisiting-network-support-for-rdma/stacks.svg)

- **iWARP** (IETF RFCs 5040 to 5044, 2007) runs RDMA over TCP. To get from RDMA messages to TCP's byte stream and back, the NIC stacks three layers on TCP:
  - **RDMAP**: the RDMA operations (Read, Write, Send);
  - **DDP** (direct data placement): carries each segment's target buffer and offset, so it can be placed in memory directly;
  - **MPA** (marker PDU aligned framing): markers and a CRC, so the receiver can find segment boundaries in the byte stream.
- **Why RoCE won.** The authors asked NIC vendors and data center operators, and heard consistently that iWARP is more complex and expensive than RoCE, with worse performance.
- **The paper's own data point** (its Table 1): 64-byte RDMA Writes on one queue pair, two NICs at 40 Gb/s.

| NIC | Price | Message rate | Latency |
|---|---|---|---|
| Chelsio T580-CR (iWARP) | $760 | 3.24 Mpps | 2.89 µs |
| Mellanox MCX416A-BCAT (RoCE) | $420 | 14.7 Mpps | 0.94 µs |

- The authors call this anecdotal: margins, features and engineering effort differ too. But it shows that their starting conjecture, that loss belongs in the NIC, was not obvious from the iWARP NICs on the market.
- **The thesis in one line.** iWARP "had the right philosophy", handling loss in the NIC instead of making the network lossless, but "a needlessly complex design". IRN starts from RoCE's simple transport and asks what is the least it must add.

## 2. IRN's design

> **Takeaway:** IRN (improved RoCE NIC) makes two changes: **selective retransmission**, where the receiver keeps out-of-order packets and the sender resends only the lost ones, and **BDP-FC**, a static cap of one bandwidth-delay product of packets in flight per QP. Congestion control stays separate and optional.

The stated goal is the fewest changes to a RoCE NIC that remove its need for PFC, not the best possible transport. Both changes are about loss and in-flight data; DCQCN or TIMELY can run on top, as with RoCE.

**ACK, NACK and SACK.** Three kinds of message a receiver sends back about the packets it got; the rest of this post uses all three.

- **ACK** (acknowledgment): "your packets arrived." It is usually **cumulative**: an ACK carrying PSN *n* means every packet before *n* has arrived and *n* is the next one expected. One ACK covers many packets, and if an ACK is lost, the next one covers for it.
- **NACK** (negative acknowledgment): "something is missing." The receiver sends one when a packet arrives out of order, beyond the PSN it expects, which suggests the expected packet was lost. In RoCE, a NACK makes the sender go back N; in IRN, it starts selective retransmission.
- **SACK** (selective acknowledgment): "this particular packet arrived," even though an earlier one is missing. A cumulative ACK cannot say that; a SACK lets the sender resend only what was actually lost. TCP carries SACKs as byte ranges; IRN puts a single PSN in each NACK.

For example, if packets 0 to 5 are sent and packet 3 is lost, the cumulative ACK stops at 3, and packets 4 and 5 each draw a NACK that also SACKs them. The sender then knows to resend 3 and nothing else.

### 2.1 Selective retransmission

![An IRN sender's packets, PSN 0 to 15. Packets 0 to 2 are ACKed and the cumulative ACK is 3, the receiver's next expected PSN. One NACK per out-of-order arrival SACKed 4, 5 and 7; 6 was lost too, so no NACK named it. Packets 3 and 6 are presumed lost, and 8 and 9 were sent with no news yet. The BDP-FC window covers PSNs 3 to 12 (a cap of 10 here, 110 in the paper), so new packets 10, 11 and 12 are allowed and 13 to 15 are blocked. The recovery sequence is 9, the last new PSN sent before the first resend; recovery ends once the cumulative ACK passes it. What goes out next, in order: 3, the cumulative ACK, always the first resend; 6, presumed lost because 7 above it was SACKed; then new packets 10, 11 and 12. 8 and 9 are not resent, because nothing above them has been SACKed. A lost packet with nothing after it gets no NACK, so a timeout recovers it: 100 µs with at most 3 packets in flight, 320 µs otherwise](/assets/images/posts/revisiting-network-support-for-rdma/sender-window.svg)

**The receiver** keeps out-of-order packets (section 4 shows how) and, for **every** out-of-order arrival, sends a NACK carrying two numbers:

- the **cumulative acknowledgment**, its next expected PSN;
- the **PSN of the packet that triggered the NACK**: a one-packet SACK.

**The sender** keeps, per QP, a bitmap of which packets have been cumulatively or selectively acknowledged, and enters **loss recovery** on a NACK or a timeout. In recovery, it works from the bitmap (above):

1. It resends lost packets instead of sending new ones, starting with the packet at the cumulative acknowledgment.
2. It presumes any other packet lost **only if a higher PSN has been SACKed**. Packets with nothing SACKed above them may still be in flight, so they are left alone.
3. With nothing left to resend, it sends new packets, if BDP-FC allows.
4. It leaves recovery when the cumulative acknowledgment passes the **recovery sequence**: the last new packet it sent before its first resend.

**Timeouts.** A NACK needs a later packet to arrive. With few packets in flight, as in a single-packet message or the last packets of a longer one, a loss may produce no NACK, and only a timeout recovers it. IRN uses two static values:

- **RTO<sub>low</sub> = 100 µs** when at most N = 3 packets are in flight: the cases where a timeout is the likely way out. It bounds their tail latency, and with so few packets out, a spurious resend costs little.
- **RTO<sub>high</sub> = 320 µs** otherwise: the largest round trip with one congested link, that is, propagation on the longest path plus a full switch buffer's queuing.

### 2.2 BDP-FC

- **The rule.** Send a new packet only while fewer than a fixed cap are in flight, counted as the new PSN minus the cumulative ACK. SACKed packets do not free room: the window slides only with the cumulative ACK, so a flow in recovery cannot pile new packets onto the congestion.
- **The cap** is one bandwidth-delay product (BDP): the bandwidth is the link rate, 40 Gb/s; the delay is the round trip of the longest path: 6 links of 2 µs each way, so 24 µs. Their product, 120 KB, is what a sender must have in flight to keep its link busy until the first ACK returns; in 1 KB packets, **about 110**. It is computed from the known topology, and overestimating it costs a little queuing, not utilization.
- **It is per QP, not per link.** Many QPs sharing a link can still overfill it: BDP-FC bounds what each flow can put into the network, but does not divide the link among flows. That is congestion control's job; without it, IRN drops 8.5% of its packets (section 3.2).
- **It fixes the NIC's state.** No packet can arrive more than one cap ahead of the receiver's next expected PSN, so **every bitmap has a fixed size per QP**: 128 bits at 40 Gb/s (section 5.1).

### 2.3 What IRN leaves out of TCP

- **No congestion window.** Classic TCP resizes its window as it goes: slow start doubles it each round trip, then AIMD adds one packet per round trip and halves it on a loss. IRN's only window is BDP-FC's fixed cap, so flows start at line rate; any rate control comes from optional DCQCN or TIMELY.
- **Packets, not bytes.** IRN numbers fixed-size RDMA packets, so acknowledgments and loss tracking are bitmaps over PSNs, and there is no byte stream to frame (no MPA, no DDP).

**My take.**

- The one-PSN SACK assumes packets mostly arrive in order (one path per QP), so NACKs come only after a loss, each in place of the ACK that packet would have drawn anyway.
- Why not TCP's SACK ranges? Simplicity, mostly: one fixed field is easy for NIC hardware, and the sender's bitmap does the bookkeeping.

## 3. Does RDMA need a lossless network?

### 3.1 The setup

> **Takeaway:** A packet-level simulator from Mellanox, modeling a ConnectX-4 RoCE NIC, on a small three-tier fat-tree under a heavy-tailed workload at 70% load. Each configuration is compared with and without PFC.

- **Simulator.** Obtained from a commercial NIC vendor; built on OMNeT++/INET (an open-source, discrete-event network simulator), it models the Mellanox ConnectX-4 and its DCQCN implementation. The authors added a NIC-based TIMELY.
- **Switches** (below). Each input port queues packets separately for each output port, and each output serves the inputs in turn. An input holds 240 KB, twice the BDP, and pauses its upstream sender above 220 KB.

    ![A switch with three inputs and three outputs, A, B and C. Each input port keeps a separate queue for each output: input 1 holds three packets for A and one for B; input 2 two for A and two for C; input 3 five for A, three for B and three for C. Every input's queue for A leads to output A, which takes one packet from input 1, 2, 3, 1 and so on, in turn. A bar under each input shows the bytes it holds against its 240 KB buffer; input 3 is past the 220 KB threshold, so it sends PAUSE to its upstream sender. Steps: 1, lookup: on arrival, the switch finds the packet's output from its destination and queues it at its input, in the queue for that output; 2, scheduling: each output takes packets from the inputs in turn (round-robin), so a busy output never blocks packets bound elsewhere; 3, PFC: each input holds 240 KB, above 220 KB it pauses the sender upstream, and the last 20 KB catches what is still on the wire](/assets/images/posts/revisiting-network-support-for-rdma/switch-model.svg)

- **Topology** (below). A fat-tree: a three-tier Clos of identical *k*-port switches, each with half its ports down and half up, so every tier carries as much up as down (full bisection bandwidth). With *k* = 6, that is 45 switches and 54 servers. Links are 40 Gb/s with 2 µs of propagation delay; the longest path crosses 6 links.

    ![A fat-tree of 6-port switches: 9 core switches on top, then 6 pods, each with 3 aggregation and 3 edge switches, and 3 servers under each edge switch. Each aggregation switch connects to 3 core switches and to every edge switch in its pod. Every switch has 6 ports, half down and half up, so there are 18 + 18 + 9 = 45 switches and 6 × 3 × 3 = 54 servers. The highlighted longest path runs from a server in pod 1 up through its edge, aggregation and core switches and down to a server in pod 6: 6 links of 2 µs, a 24 µs round trip, and at 40 Gb/s a BDP of 120 KB](/assets/images/posts/revisiting-network-support-for-rdma/fat-tree.svg)

- **Workload.** Poisson flow arrivals to random destinations at 70% average link load, spread by ECMP, with heavy-tailed sizes:
  - 50% of flows are single packets of 32 B to 1 KB, like the RPCs of RDMA key-value stores;
  - 15% are 200 KB to 3 MB, like storage traffic, and carry most of the bytes.
- **Timeouts.** IRN as in section 2.1. RoCE without PFC uses a fixed 320 µs; with PFC, timeouts are disabled to avoid spurious resends.
- **Metrics.**
  - **Average slowdown**: a flow's completion time divided by its time alone on an empty network. Dominated by short flows.
  - **Average and 99th percentile flow completion time (FCT)**. Dominated by large flows.

### 3.2 Three results

> **Takeaways:**
> - IRN without PFC beats RoCE with PFC, by 2.8 to 3.7 times.
> - PFC makes IRN **worse**, by 1.5 to 2 times.
> - RoCE without PFC is worse again, by 1.5 to 3 times. RoCE needs PFC; IRN does not.

![Three horizontal bar charts for the default scenario without congestion control; lower is better. Average slowdown: RoCE without PFC about 75, RoCE with PFC 30.6, IRN with PFC 16.1, IRN without PFC 8.2. Average FCT: about 8.2 ms, 2.6 ms, 1.4 ms and 0.9 ms. 99th percentile FCT: about 81 ms, 50.8 ms, 25.0 ms and 15.3 ms. IRN without PFC drops 8.5% of its packets and still wins on every metric; turning PFC on makes IRN worse by 1.5 to 2 times and RoCE better](/assets/images/posts/revisiting-network-support-for-rdma/results.svg)

**IRN does not need PFC; it is better off without it.**

- Without PFC, IRN loses **8.5%** of its packets, and each loss costs about a round trip to detect and another to repair.
- But a drop hurts only the flow that met the congestion. A pause hurts every flow behind the paused port, and pauses cascade upstream.
- In the paper's words, "a well-design[ed] loss-recovery mechanism outweighs a lossless network."

### 3.3 With congestion control

> **Takeaway:** With TIMELY or DCQCN, PFC hardly matters to IRN in the default scenario (under 1% better at best, 3.4% worse at worst), still matters to RoCE (1.35 to 3.5 times), and IRN still beats RoCE with PFC by 1.5 to 2.2 times.

| Average slowdown / 99th percentile FCT | with TIMELY | with DCQCN |
|---|---|---|
| RoCE, no PFC | ≈25 / ≈72 ms | ≈14 / ≈67 ms |
| RoCE + PFC | 7.6 / 31 ms | 10.7 / 46 ms |
| IRN + PFC | 4.8 / 19 ms | 5.1 / 21 ms |
| **IRN, no PFC** | **4.7 / 18.5 ms** | **5.2 / 20.7 ms** |

### 3.4 Which change matters more

> **Takeaway:** Both count, but loss recovery counts more. The cheaper alternatives the authors tried all did worse, or no better.

| Average FCT (ms) | IRN | IRN with go-back-N | IRN without BDP-FC |
|---|---|---|---|
| no congestion control | 0.9 | 7.1 | ≈2.3 |
| with TIMELY | 1.2 | ≈1.8 | ≈1.5 |
| with DCQCN | 1.0 | ≈1.5 | ≈1.5 |

- **Go-back-N** wastes bandwidth on duplicates: 8 times slower without congestion control.
- **Without BDP-FC**, queues grow, and a flow in recovery keeps adding new packets.
- **Alternatives** to IRN's loss recovery:
  1. **A gentler go-back-N**, backing off explicitly on loss: helped with TIMELY, not with DCQCN, and SACK-based recovery was still 20 to 50% better in average FCT.
  2. **Selective retransmission without the SACK bitmap**: better than go-back-N, but several losses in one window take several round trips; <1% to 75% worse than IRN.
  3. **Adaptive timeouts**, computed from measured round trips as in TCP: more complex, and no better, since results were dominated by the initial value.

### 3.5 Robustness

> **Takeaway:** Across load, link speed, scale, workload, buffer size and timeout settings, IRN without PFC always beats RoCE with PFC, by 6 to 83%, by the paper's count. PFC hurts IRN most at high load, low link speed and small buffers.

How much longer the average flow takes than with IRN without PFC (no congestion control; my ratios from the extended version's appendix tables):

| Scenario | IRN + PFC | RoCE + PFC |
|---|---|---|
| 30% load | 1.00× | 1.35× |
| 50% load | 1.07× | 2.9× |
| 70% load (default) | 1.56× | 2.9× |
| 90% load | 1.75× | 2.1× |
| 10 Gb/s links | 1.91× | 3.2× |
| 100 Gb/s links | 1.37× | 2.1× |
| 128 servers / 250 servers | 1.66× / 1.66× | 3.0× / 3.1× |
| uniform flow sizes, 500 KB to 5 MB | 3.0× | 4.3× |
| 60 KB buffer per port | 2.7× | 2.2× |
| 480 KB buffer per port | 1.05× | 3.2× |

- **The trends.** PFC hurts IRN more as load rises (more congestion spreading), less as links get faster (a drop's round trip costs relatively more), and more as buffers shrink (more pauses). With 480 KB per port, PFC rarely fires, so it costs IRN little, while IRN's lead over RoCE grows.
- **Window-based congestion control** (TCP's AIMD, DCTCP) shows the same trends. With AIMD, PFC hurts even more, since it hides the drops that AIMD uses as its congestion signal.

### 3.6 Against iWARP's TCP

> **Takeaway:** On the same simulator, IRN without congestion control matches a full TCP stack on flow completion time and beats it on slowdown.

- Against INET's TCP implementation, standing in for iWARP's transport: average slowdown 8.2 against ≈10.4 (21% lower), with comparable average and tail FCT. The difference is slow start, which IRN replaces with BDP-FC.
- IRN with TCP's AIMD added beats it by 44% on slowdown and 11% on average FCT.
- This compares transport logic only. The hardware argument is section 5: IRN keeps RoCE's message rate, while an iWARP NIC can be 4 times slower (section 1.3).

## 4. Making it work in a RoCE NIC

> **Takeaway:** The hard part is not the retransmission logic but out-of-order delivery: RoCE's headers and completion rules assume that packets arrive in order. IRN fixes this with bitmaps, a few header fields and sequence numbers on work requests, all inside the UDP payload, so switches do not change.

### 4.1 The parts of RoCE that assume order

**Vocabulary.** The two ends of a transfer are the **requester** and the **responder**. An application posts a work queue element (WQE, "wookie") for each message; WQEs at the requester are Request WQEs, those at the responder Receive WQEs. When a message completes, its WQE expires and a completion queue element (CQE, "cookie") tells the application.

| Operation | What it does | Receive WQE at the responder |
|---|---|---|
| Write | requester writes into the responder's memory; source and target addresses in the Request WQE | none (Write-with-immediate consumes one, to notify the responder) |
| Read | requester reads the responder's memory | none |
| Send | requester sends; the responder's Receive WQE says where the data lands | one per message, consumed in posting order |
| Atomic | read and update one location, in a single packet | none |

**Who acknowledges what.** In RoCE, the requester alone is responsible for reliability: it keeps the timers and does the resending, and the responder only answers.

- **When the requester sends data (Writes, Sends), the responder ACKs it**, per packet in RoCE NICs. A Receive is not an operation on the wire, only the responder's buffer for an incoming Send, so the Send's ACKs cover it.
- **When the requester receives data (Reads), the data itself is the ACK.** The Read responses coming back acknowledge the Read request, so no explicit ACK is needed in either direction. If a response goes missing, the requester notices the gap in their PSNs and issues the Read again from the missing packet; the responder keeps no retransmission state and simply reads its memory again.

**What breaks once packets can arrive out of order:**

- **Critical information rides in one packet of a message.**
  - *First packet.* A Write's target address and length (its RETH header) travel only in its first packet. The responder remembers them, and places each later packet at the next offset. A middle packet that arrives first has nowhere to go.
  - *Last packet.* The last packet of a Write or Send advances the responder's message sequence number (MSN). Like the PSN, the MSN belongs to InfiniBand's transport layer (L4): it travels back in the ACK header (AETH) and lets the requester retire its Request WQEs. The last packet also carries what goes into the CQE, such as a Write-with-immediate's immediate data. Arriving early, it would complete a message whose earlier packets are still missing.
- **Receive WQEs are matched by position.** A Send packet names its QP (the QPN in its header), but not which Receive WQE it is for, nor where in that buffer it belongs. Within an RC QP (the reliable, connected kind IRN targets), the responder takes the next Receive WQE, in posting order, when a Send's first packet arrives, and fills it at a running offset as the rest arrive. In order, "the next Send to arrive" is "the next Send sent", so the *n*-th Send lands in the *n*-th Receive WQE. (QPs never mix: each has its own receive queue, unless several share one, a shared receive queue.)
- **Applications rely on it too.** FaRM uses plain RDMA Writes, which tell the responder's application nothing, so its CPU busy-polls memory instead. Each message ends with a trailer that stays zero until written; once it turns non-zero, the whole message is taken as written, because the NIC writes in increasing address order. That holds only while the last packet lands last. (Write-with-immediate would signal completion properly, but costs a Receive WQE and a CQE per message.)
- **One PSN space per requester.** One counter numbers both the requests it sends and the Read responses it receives: a 3-packet Write takes PSNs 10 to 12, a 4-packet Read's request takes 13 and its responses 13 to 16, and the next Send gets 17. Counting in-flight packets as "PSN minus acknowledged PSN" then mixes both directions.

### 4.2 Acknowledging Reads

- IRN's selective retransmission and BDP-FC run where the data comes from, which for a Read is the responder. So IRN adds Read (N)ACKs from the requester, using one of the eight opcodes still unused for reliable connections, and gives the Read responder a retransmission timer.
- Atomics are treated as single-packet Reads.

### 4.3 Accepting packets out of order

![A 4-packet RDMA Write, PSNs 20 to 23; PSN 21 is lost on its first try. Packets arrive in the order 20 (first), 22, 23 (last) and, a round trip later, 21 (resent); each carries its address in a RETH header. Each is DMAed straight to its own slot in the application buffer at address A, at A plus 0, 1, 2 and 3 KB, with no reordering buffer. In the NIC, a 2-bitmap starts at the next expected PSN, 21: the arrived row has 22 and 23 set, and the ends-message row has 23 set. 23 ends the message, but the MSN update and the completion wait for 21. Buffering out-of-order packets on the NIC instead would take up to 110 packets of 1 KB, 110 KB per QP or 110 MB for 1,000 QPs; the bitmap costs 2 bits per packet, 256 bits per QP at 40 Gb/s](/assets/images/posts/revisiting-network-support-for-rdma/ooo-placement.svg)

- **The naive way** buffers out-of-order packets on the NIC until the gap fills: up to 110 packets of 1 KB per QP, or 110 MB for a thousand QPs. Far more than a RoCE NIC's memory.
- **IRN's way** writes each packet straight to its final address in application memory, and records its arrival in a bitmap sized to the BDP cap: **2 bits per packet instead of 1 KB**.
- ConnectX-5 NICs already placed packets out of order, for adaptive routing, but only for Writes and Reads. IRN extends this to every operation, by solving four problems:

| Problem | Example | IRN's fix |
|---|---|---|
| First packet | Only a Write's first packet carries its target address (RETH) | Put the RETH in every Write packet (16 bytes) |
| Last packet | The MSN update, WQE expiry and CQE are triggered by a message's last packet | A **2-bitmap** at the responder: "arrived", plus "ends a message". Act on a last packet only when everything before it has arrived. A CQE prepared early waits in host memory |
| WQE matching | Sends and Write-with-immediate consume Receive WQEs in posting order; a Read or Atomic request must wait for all earlier packets | Number the WQEs. Send packets and the last Write-with-immediate packet carry a `recv_WQE_SN`, and Send packets also carry their offset in the message. Read and Atomic requests carry a `read_WQE_SN`, which picks their slot in the Read WQE buffer, where they wait |
| Application | FaRM polls the last bytes of a Write to detect completion; a resent packet of an old Write can overwrite newer data | Not supported: the RDMA specification never promised that the last bytes land last (section o9-20), and Write-with-immediate, the supported way, still works. Overwrites are left to application fences, as iWARP and ConnectX-5 already do, or to the driver |

### 4.4 Other changes

- **Two PSN spaces at the requester**: sPSN for the requests it sends and rPSN for the responses it receives, so that each can be tracked and capped. Applications do not see it.
- **Error NACKs**, such as receiver not ready, still trigger go-back-N; only out-of-sequence NACKs are treated as normal.
- Shared receive queues, Send-with-invalidate (which needs a fence) and RoCE's end-to-end credits all keep working (the extended version's appendix B).

## 5. What it costs

> **Takeaway:** About **800 bits, or 100 bytes, more state per QP** (3 to 10% of a NIC's cache for a couple of thousand QPs), a little bitmap logic that keeps up with line rate, and at worst 4 to 7% slower end to end.

### 5.1 NIC state

| What IRN adds | Size |
|---|---|
| Transport state per QP: the PSN to resend next, the recovery sequence, flags | 52 bits at each end: 104 bits |
| Read timer and in-progress Read tracking, at the responder | 56 bits |
| Bitmaps per QP, each 128 bits (one bit per packet in the BDP window): the responder's 2-bitmap counts as two, plus one for Read responses at the requester and one for SACKs at each end | 5 × 128 bits: 640 bits |
| Per WQE: the new sequence numbers | 3 bytes, on top of the current 64 |
| Shared by all QPs: the BDP cap, RTO<sub>low</sub>, N | 10 bytes |

For a couple of thousand QPs and tens of thousands of WQEs, that is **3 to 10% of a RoCE NIC's few MB of cache**, even for 100 Gb/s links.

**The five bitmaps, one by one.** Each is a 128-bit window over PSNs (enough for the BDP cap at 40 Gb/s and a 24 µs round trip), kept as a ring buffer whose head is the oldest packet not yet settled; as packets settle, the head moves on and the freed bits are reused for newer PSNs. The window covers packets, not messages: a 1,000-packet Write simply streams through it, and BDP-FC guarantees that no packet arrives more than one window ahead of the head.

- **"Arrived" (responder).** One bit per request packet it receives: Write and Send data, Read and Atomic requests. A packet sets its bit when it arrives. The first unset bit is the next expected PSN, which goes back as the cumulative ACK; a packet that arrives beyond an unset bit triggers a NACK.
- **"Ends a message" (responder).** Set when the arriving packet ends a message: the last packet of a Write or Send, or a one-packet Read or Atomic request. Nothing happens yet. Only when "arrived" slides past this bit, so that every earlier packet is in, does the message complete: the MSN goes up by one, and a Send or Write-with-immediate gets its CQE. With long messages this bitmap is mostly zeros, but a run of one-packet messages sets every bit; a list of end PSNs would then need up to 128 entries of 24 bits, so the 128-bit bitmap is the cheap choice.
- **Read responses (requester).** The requester's own "arrived" bitmap, for the one kind of data it receives. Say it issues a Read for 4 packets, PSNs 13 to 16. Each response is written straight into its buffer and sets its bit. If 14 is lost, the bitmap shows 13 and 15 once 15 arrives; the requester sends a Read NACK (cumulative 14, SACK 15), and the responder resends just 14. Plain RoCE would drop 15 and issue the Read again from 14. Once 13 to 16 are all set, the Read is complete. No "ends a message" twin is needed: the requester issued the Read, so it already knows its last PSN is 16.
- **SACKs (requester).** For the Writes and Sends it sends: a bit is set when a NACK SACKs that packet. With the cumulative ACK, it decides what to resend: the packet at the cumulative ACK first, then any unset packet below a set one (section 2.1).
- **SACKs (responder).** The same, for the Read responses it sends. For a Read the responder is the sender, and the requester's Read NACKs fill this bitmap in.

### 5.2 Logic and performance

- **The logic is small.** It is a handful of bitmap operations: find the first missing packet, count the messages that just completed, slide the window. Hardware does each in a few clock cycles. Synthesized on an FPGA, it took a few percent of the chip and kept up with more packets per second than the authors' RoCE NIC could send, so it would not slow the NIC down.
- **The end-to-end cost is small.** Even with worst-case overheads (a 2 µs fetch from host memory before every resend, and a 16-byte address header on every packet), IRN was only 4 to 7% slower than without them, and still 35 to 63% better than RoCE with PFC.
- **Vendors agreed.** Two NIC vendors, Mellanox among them, confirmed that IRN could be built in their NICs, and "Mellanox is implementing a version of IRN in their next release."

## 6. After IRN

> **Takeaway:** IRN's mechanisms became standard: selective repeat now ships in NVIDIA's RoCE NICs, and every newer lossy transport places packets out of order. But the largest RoCE deployments on commodity NICs still run PFC.

### 6.1 In RoCE NICs, and in RoCE networks

- **NVIDIA.** ConnectX-7's [firmware release notes](https://networking-docs.nvidia.com/connectx7fwrn/28381002/changes+and+new+feature+history) list "Selective Repeat" under firmware 28.37.1014: it "improves network utilization in case of a lossy fabric" and is "enabled by default". I have not found documentation of how closely it follows IRN.
- **Operators kept PFC anyway.** At Azure (NSDI'23), about 70% of the traffic in its public regions is RDMA, on PFC and DCQCN; Meta's AI training fabric (SIGCOMM'24) runs on PFC alone ([previous notes, section 7.1](/blogs/rdma-over-commodity-ethernet.html#71-lossless-roce-kept-spreading)). Section 3.3 suggests why this is tenable: with congestion control on, PFC costs at most 3.4% there, because it rarely fires. What it still costs is in operations.

### 6.2 In new transports

Two topics IRN left open, multipath and NIC scale, became the next papers' main subjects:

- **SRNIC** ([NSDI'23](https://www.usenix.org/conference/nsdi23/presentation/wang-zilong), HKUST and ByteDance) starts from IRN: it scales with the network, but not with the number of connections. SRNIC co-designs the protocol and the NIC so that selective repeat needs almost no on-chip memory, and keeps 10K connections on chip.
- **AWS SRD** ([IEEE Micro, 2020](https://ieeexplore.ieee.org/document/9167399)) sprays each flow's packets over many paths and delivers them out of order, with retransmission in AWS's own NICs.
- **Google 1RMA** ([SIGCOMM'20](https://research.google/pubs/pub49377/)) takes the opposite route to IRN: instead of adding transport logic to the NIC, it makes the NIC connection-free and leaves congestion control, failure recovery and ordering to software.
- **Google Falcon** ([SIGCOMM'25](https://dl.acm.org/doi/10.1145/3718958.3754353)) accepts lossy networks and out-of-order delivery in hardware ([my notes](/blogs/os/rocev2.html#2025)).
- **Ultra Ethernet** 1.0 (June 2025) offers reliable **unordered** delivery, which allows packet spraying, alongside an ordered mode for RoCE-like semantics, and is designed to run without PFC ([overview](https://arxiv.org/abs/2508.08906)).

## 7. Final remarks

- **What it showed.** Once a NIC can afford the state for selective retransmission, PFC is no longer a necessity, and lossy RDMA becomes realistic. Losslessness was compensating for weak loss recovery.
- **How it argued.** By finding the smallest change that removes the dependency, and pricing it: two mechanisms, a few bitmaps and header fields.
- **What it could not settle.** Production. With good congestion control, PFC costs little performance, so operators on commodity NICs (Azure, Meta) kept it; those who build their own NICs (AWS, Google) went lossy, and Ultra Ethernet followed them.
- **Read with the Microsoft paper.** That paper showed what a lossless network costs to run; this one showed that RDMA never needed one, only RoCE's NICs did.

## References

1. Radhika Mittal et al. [Revisiting Network Support for RDMA](https://dl.acm.org/doi/10.1145/3230543.3230557). SIGCOMM 2018. [Extended version](https://arxiv.org/abs/1806.08159), arXiv:1806.08159.
2. Chuanxiong Guo et al. [RDMA over Commodity Ethernet at Scale](https://dl.acm.org/doi/10.1145/2934872.2934908). SIGCOMM 2016.
3. Yibo Zhu et al. [Congestion Control for Large-Scale RDMA Deployments](https://dl.acm.org/doi/10.1145/2829988.2787484) (DCQCN). SIGCOMM 2015.
4. Radhika Mittal et al. [TIMELY: RTT-based Congestion Control for the Datacenter](https://dl.acm.org/doi/10.1145/2785956.2787510). SIGCOMM 2015.
5. Alexander Shpiner et al. RoCE Rocks Without PFC: Detailed Evaluation (Resilient RoCE). KBNets 2017.
6. Renato Recio et al. [A Remote Direct Memory Access Protocol Specification](https://www.rfc-editor.org/rfc/rfc5040) (iWARP RDMAP). RFC 5040, 2007.
7. [NVIDIA ConnectX-7 Firmware Release Notes v28.38.1002](https://networking-docs.nvidia.com/connectx7fwrn/28381002/changes+and+new+feature+history): Changes and New Feature History.
8. Wei Bai et al. [Empowering Azure Storage with RDMA](https://www.usenix.org/conference/nsdi23/presentation/bai). NSDI 2023.
9. Adithya Gangidi et al. [RDMA over Ethernet for Distributed AI Training at Meta Scale](https://dl.acm.org/doi/10.1145/3651890.3672233). SIGCOMM 2024.
10. Zilong Wang et al. [SRNIC: A Scalable Architecture for RDMA NICs](https://www.usenix.org/conference/nsdi23/presentation/wang-zilong). NSDI 2023.
11. Leah Shalev et al. [A Cloud-Optimized Transport Protocol for Elastic and Scalable HPC](https://ieeexplore.ieee.org/document/9167399) (SRD). IEEE Micro, 2020.
12. Arjun Singhvi et al. [1RMA: Re-envisioning Remote Memory Access for Multi-tenant Datacenters](https://research.google/pubs/pub49377/). SIGCOMM 2020.
13. [Falcon: A Reliable, Low Latency Hardware Transport](https://dl.acm.org/doi/10.1145/3718958.3754353). SIGCOMM 2025.
14. [Ultra Ethernet's Design Principles and Architectural Innovations](https://arxiv.org/abs/2508.08906). arXiv:2508.08906, 2025.
