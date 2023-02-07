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
Lazy update the temperature of the sampled pages; different threshold for load and store
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

Maintain freshness with lazy update:
```
threshold: 18 accesses
When to update: set a global clock, if any of the sampled pages reach the threshold, global clock++
How to update: the next time a page is sampled, if local clock does not match global clock, cut half the access counts
```

### HeMem Library Mechanisms
```
allocation: intercept system calls and allocate memory via DAX files and according to its policy
memory migration: carried out by policy thread; with the help of userfaultfd and DMA engine
page fault handling: page fault + write-procetion fault
```
a) allocation

intercept `mmap` system calls and other C library functions;

focus on application heap ranges that have the highest tendency to grow and live longer;

allocate DRAM and NVM via DAX files; maps per-process DAX files into virtual address;

Upon mmap, HeMem allocates memory regions according to its policy;

HeMem then tracks the mapping from virtual address to file offset for each page it manages;

b) memory migration

carried out by a `plicy` thread;

migration: `userfaultfd` mark page as write-protected ==> migrate with I/OAT DMA engine ==> restore access rights

- about write-protected: write pauses due to migration is exceedingly rare
- about DMA engine: 

i) DMA data copy API via ioctl calls: src va, dest va, size, DMA channel identifiers

ii) can handle multiple copy requests in batches

iii) can use multiple DMA channels to copy in parallel

c) Page fault handling

register memory allocation with `userfaultfd` and accept page faults, as well as write-protection faults;

fault handling thread reads userfaultd file descriptor;

### HeMem Policies
```
Memory allocation and placement: DRAM first, then NVM; small, ephemeral reside in DRAM
Memory migration: write-heavy pages are more prioritized
```
a) Memory allocation and placement

allocate DRAM if available(remove a page from the free list); ephemeral data remain in fast DRAM;

then allocate NVM, and rely on PEBS thread to tell migrate candidates;

policy keeps 1GB DRAM free, and force migration at this threshold;

HeMem forward intercepted memory allocation calls with small memory allocation to the kernel; otherwise it manages by itself

b) Memory migration

Migrate among DRAM cold list and DRAM hot list;

Write-heavy pages are given higher priority for migration to DRAM than read-heavy pages, due to NVM having lower write than read performance;

write-heavy pages are moved to the front of the hot list;

second chance for write heavy pages: write heavy pages is moved to the hot list during cooling.
