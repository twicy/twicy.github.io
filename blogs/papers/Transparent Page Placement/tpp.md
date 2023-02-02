# TPP: Transparent Page Placement for CXL-Enabled Tiered Memory

## Background

- Increased Memory Demand in Datacenter Applications

- Scaling Challenges in Homogeneous Server Designs
Memory bandwidth and capacity does not increase proportionally.
High coupling of CPU and memory subsystem strands resources.

- CXL for Designing Tiered Memory Systems
In a word: CXL looks like a CPU-less NUMA node from the perspective of a software.
CXL is an industry-supported interconnect based on the PCI Express (PCIe) interface.
CXL enables high-speed, low latency communication between the host processor and devices.
CXL-Memory access latency is also similar to the NUMA access latency.

- Scope of CXL-Based Tiered Memory Systems
The author characterizes 4 popular applications in Meta's fleet.

## Major Contributions

- Chameleon, a lightweight user-space memory characterization tool. Note that Chameleon is an offline tool, whose purpose is to gain prior knowledge about the production workload's memory consumption behavior.
- TPP, a kernel machinism for efficient memory management on tiered-memory system

## Design

### Chameleon
In a word, the chameleon utilizes the PEBS mechanism of a CPU's PMU to collect the temperature of pages that triggers specific events.
- Collector
Collect virtual address and PID of pages that triggers: MEM_LOAD_RETIRED.L3_MISS and MEM_INST_RETIRED.ALL_STORES.

To leverage accuracy and overhead, one sample for every 200 events is a good tradeoff.[sample_period = 200]

To avoid CPU stalling, rotate to a divided CPU core group.
- Worker
Track the temperature of all the pages reported by the collector with a 64-bit bitmap

### Production Workload Overview
- A significant portion of a datacenter application’s accessed memory remain cold for minutes. Tiered-memory system can be a good fit for such cold memory.
- A large fraction of anon pages tend to be hot, while file pages remain cold within short intervals.
- Although anon and file usage may vary over time, applications mostly maintain a steady usage pattern.
- Workloads have different levels of sensitivity toward different memory types.
- Cold page re-access time varies for workloads. Page placement on a tiered memory system should be aware of this to avoid high memory access latency.

### Problems with Linux Page Placement Policy
- Original Linux is designed for homogenous DRAM-only NUMA nodes.
- Page allocation and reclamation are tightly coupled

Memory Zone maintains: min, low, high watermark

Frees inactive file cahes, anon, mapped file pages

Only after enough memory has been freed to satisfy local memory node’s high_watermark, will local allocation happen.

Due to the fact that relcamation is slower that allocation, local memory halts frequently and degrade application performance.







