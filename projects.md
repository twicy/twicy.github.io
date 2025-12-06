---
layout: page
permalink: /projects/index.html
title: Projects
---

### SGEMM + Two Minimum Value and Index Searching -- (2025.09 - 2025.12)

Stony Brook University, Coursework of [ESE 565: Parallel Processing Architectures](http://www.ece.sunysb.edu/~midor/ESE565/index.html) Fall 2025, Advisor: [Prof. Mikhail Dorojevets](https://www.ece.stonybrook.edu/~midor/)

- **Requirement**: Perform a matrix multiplication of two randomly generated floating-point square (I generalized it to **any arbitrary size**) matrices, then find the **the two minimum values and their indices** in the product matrix, with sequential C/C++, ISPC (with multi-task), Pthread (with ISPC), OpenMP (with ISPC), 1/2 GPU CUDA, 4 GPU CUDA with MPI
  - **Challenge 1**: **Maximize parallelization** of minimum searching in each subroutine (ISPC instance/Pthread/OpenMP thread/CUDA thread block), instead of reducing it sequentially in the main route.
  - **Challenge 2**: Atomic update of both minimum **value and index**
- I took cache (locality and false sharing), SIMD/thread/task/SIMT parallelism into consideration achieving the following result, compared with naive Sequential C:
  - ISPC with multi-task: **199×**
  - Pthread/OpenMP with ISPC: **259×**
  - 2-GPU CUDA: **16388×**
  - 4-GPU CUDA MPI: **667×**
- Specifically, my CUDA program achieved $80\% - 85\%$ of cuBLAS's performance on matrices of order 2k, 4k, 8k
  - Hierarchy: Device level, Thread Block level, Warp level, Thread level
  - Vectorized load, Bank conflict, Coalesced memory access
  - Double buffering
  - Works on **arbitrary sizes**
- Archived as an example of an outstanding ESE 565 project

  ![The memory and thread hierarchies in the CUDA programming model.](/images/projects/gpu_arch.png)
  Image source: [Strassen's Algorithm Reloaded on GPUs](https://dl.acm.org/doi/10.1145/3372419)

### CXL-based VM Live Migration -- (2025.04 - 2025.06)

Stony Brook University, Advisor: [Prof. Erez Zadok](https://www3.cs.stonybrook.edu/~ezk/), [Dr. Tyler Estro](https://www.fsl.cs.stonybrook.edu/~tyler/), [Mr. Daniel Berger](https://www.microsoft.com/en-us/research/people/daberg/), [Prof. Michael Ferdman](https://compas.cs.stonybrook.edu/~mferdman/), [Prof. Geoff Kuenning](https://scholar.google.com/citations?user=wVjW_1MAAAAJ&hl=en)

- **Background**: VM live migration is a widely utilized technique in data centers for system upgrade, load balancing, etc. Existing VM live migration leverages TCP/IP or RDMA as its underlying data transmission channel, **inevitably having a data copy at the `dst` side**, which is widely deemed as a price that must be paid.
- However, this axiom-like assumption is now **overturned** by hardware memory pooling, provided by commodity-ready [Compute Express Link (CXL)](https://computeexpresslink.org/) 2.0.
- [Mohit](https://www.linkedin.com/in/mohit-kumar-verma-a94962150/) and I implemented a working proof-of-concept prototype on emulated CXL hardware, achieving a **2×** faster migration speed over traditional TCP-based methods in internal benchmarks
- We both helped implement QEMU-level CXL tiering solution
  - Only hot pages are transferred
  - Cold pages are remapped rather than migrated
  - **Zero-copy** by definition

  ![CXL2.0 Memory Pooling](/images/projects/cxl2.0_memory_pool.png)
  Image source: [Compute Express Link™ 2.0 Specification: Memory Pooling](https://computeexpresslink.org/wp-content/uploads/2023/12/CXL-2.0-Memory-Pooling.pdf)

### Ethernet Network Driver Development for AMD AXI FPGA -- (2024.10 - 2025.03)

Stony Brook University, Advisor: [Prof. Michael Ferdman](https://compas.cs.stonybrook.edu/~mferdman/)

- [Mohit](https://www.linkedin.com/in/mohit-kumar-verma-a94962150/) and I implemented both TX and RX logic from scratch, 900+ lines of clean, well-documented kernel code, supporting multiple features
  - Multi-fragment TX transmission
  - In-place RX reception
  - NAPI-based polling
- Manipulated two ring buffers of hardware descriptors and registers to ensure safe DMA operations, properly handling erroneous situations and fringe cases
- Achievements:
  - $50\%$ performance increase compared with last generation under iperf3 test suite
  - We are the [first students in the past 5 years](https://www.linkedin.com/posts/yuchen-tang-b49a37190_thrilled-to-share-that-our-joint-work-on-activity-7320867195838701570-IxKV?utm_source=share&utm_medium=member_desktop&rcm=ACoAAC0A4KkB_ikcsCKVg_BEXlplJZeRxyaKkiU) to actually deliver a steady and high-performance driver
  - It has been running smoothly and bug-free for **over six months**, reliably serving as core infrastructure for the [COMPAS lab](https://compas.cs.stonybrook.edu/) — something its predecessor had **NEVER** achieved.

  ![Instructor comment](/images/projects/nic_instructor_comment.jpeg)

### RDMA communication middleware (OpenUCX) optimization --  (2023.10 - 2024.05)

Huawei Technologies Co., Ltd, Advisor: [Dr. Yunfei Du](https://scholar.google.com/citations?user=6vf_uwYAAAAJ&hl=en&oi=ao), Tuo Fang, [Dr. Yuxin Ren](https://orcid.org/0000-0003-2678-9225), [Prof. Guyue (Grace) Liu](https://scholar.google.com/citations?user=9NPX8KMAAAAJ)

- [OpenUCX](https://openucx.org/) is an open-source, production-grade communication framework for data-centric and high-performance applications.
- Helped implement **XRC service type**, efficiently bringing down QP numbers thus increasing RDMA scalability
- Helped resolve the **Head-of-Line blocking** problem with a simple yet effective two-level message slicing policy
- Helped set up a finer **credit-based flow control** mechanism
- Evaluated our implementation based on both micro-benchmarks and real-world applications
- Our work has been recognized and the paper has been accepted to **EuroSys'26**

  ![OpenUcx Layout](/images/projects/openucx_layout.jpg)

### Memory node expansion utilizing FPGA-based CXL 1.1 devices --  (2023.04 - 2024.12)

- **Background**: To meet the need for more memory from HPC and database applications, our lab researched into tiered memory systems, where ideally we exploit a fast DRAM tier for performance and a slow SSD/NVM tier for capacity.
- However such systems need tuning and clever data placement strategies to avoid severe application performance penalty. [etmem](https://github.com/openeuler-mirror/etmem), is Huawei's tiered memory system manager. It takes a traditional approach of **access bit checking and watermark comparison** to determine the temperature of a page.
- [Compute Express Link (CXL)](https://computeexpresslink.org/) is a new CPU to peripheral interconnect. With its **CXL.mem** subprotocol, CPUs are able to access CXL memory as cacheable system memory. With access latency **comparable to that of a remote NUMA node** (~100ns for 64B read), it has been one of the most promising emerging memory extent spotted these years.
- I observed the frequent **page ping-pong** caused by original, **coarse-grained access bit checking** method, and developed a substitute kernel module that multiplexed the original etmem workflow, selectively listening to either access bit checking or fine-grained hardware access counter results.

  ![Memory Hierarchy](/images/projects/memory_tier.png)
  Image source: [TPP: Transparent Page Placement for CXL-Enabled Tiered-Memory](https://arxiv.org/abs/2206.02878)
