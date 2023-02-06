# HeMem: Scalable Tiered Memory Management for Big Data Applications and Real NVM

## Background
- Application Memory Demands
- Intel Optane DC Non-volatile Memory

Optane is more sensitive to DRAM[asymmetric read/write bandwidth; access granularity larger than a cache line, wears out faster than DRAM]

writing to Optane is slow compared to DRAM

Accessing small objects(less than 4KB) randomly on Optane is slow

There's spare memory bandwidth available to migrate data among tiers of memory

- Traditional page table scanning method has limited power at TB level
- NVM integrated into a tiered memory system in three ways

a) Memory mode; DRAM serves as a cache of NVM

Does not distinguish the temperature of a page; 

Conflict misses[multiple physical addresses are mapped to the same location in the DRAM cache] increase evictions and reduce cache efficiency;

If a cache line is modified, eviction incurs write-back, causing more random writes to NVM and reduce perfomance;

b) Kernel-managed NUMA memory; expose as a remote NUMA node

c) File system; application map files to memory and access by processor load and store


## Major Contributions

a) analyze the performance of Intel Optane DC

b) HeMem, a tiered main memory management system

c) Evaluate HeMem, and compare with other models: hardware-based, Nimble Linux, X-mem
## Design

### Memory Access Measurement
```
Asynchronous PEBS sampling thread
```
a) Asynchronous memory access sampling

PEBS sampling scales with memory size; 
```
Configuration:
sample event: all loads from NVM `MEM_LOAD_RETIRED.LOCAL_PMM`, all loads from DRAM `MEM_LOAD_L3_MISS_RETIRED.LOCAL_DRAM`, all stores `MEM_INST_RETIRED.ALL_STORES`
sample data: virtual address
sample_period: 5000
```

b) data classification
(HOT, COLD) X (DRAM, NVM), 4 lists in total

If the sample is a part of a managed memory region, increment access count;

Separate counter for read and write(load and store); 8 loads or 4 writes makes a page hot; pages with >= 4 stores are "write-heavy"

### HeMem Library Mechanisms
### HeMem Policies
