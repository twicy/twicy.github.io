---
layout: page
permalink: /projects/index.html
title: Projects
---

### Ethernet Network Driver Development for AMD AXI FPGA -- (2024.10 - 2025.1)

Stony Brook University, Advisor: [Prof. Michael Ferdman](https://compas.cs.stonybrook.edu/~mferdman/)

- Implemented both packet transmission and reception logic from scratch, with multiple features
  - 900+ lines of kernel code
  - Support multi-fragment packet transmission, zero copy and NAPI-based polling
- Manipulated two ring buffers of hardware descriptors to ensure safe and complete DMA operations
- Was the first student in the past 5 years to actually deliver a steady and high-performance driver, which has become part of the infrastructure of the lab
  - 50\% performance increase compared with last generation under iperf3 test suite
  - The source code is now more human-readable and maintainable, reducing lines of code from over 4,100 to around 1,200 while adhering to the standard Linux Kernel coding style.

### OpenUCX Optimization --  (2023.10 - 2024.05)

Huawei Technologies Co., Ltd, Advisor: [Dr. Yunfei Du](https://scholar.google.com/citations?user=6vf_uwYAAAAJ&hl=en&oi=ao), [Dr. Yuxin Ren](https://orcid.org/0000-0003-2678-9225)

- [OpenUCX](https://openucx.org/) is an open-source, production-grade communication framework for data-centric and high-performance applications.
- It is based on a set of abstract communication primitives, inclusing RDMA. However, current implementation of openucx is not enough, as it fails to mitigate the widely known scalability issue of RDMA network.
- Helped implementing XRC service type, efficiently bringing down QP numbers thus increasing RDMA scalability.
- Helped resolving the Head-of-Line blocking problem with message slicing and finer credibility-based priority mech
anism.
- Evaluated our implementation based on both micro-benchmarks and real world applications.

![](/images/projects/openucx_layout.jpg)

### Memory node expansion utilizing FPGA-based CXL 1.1 devices --  (2023.04 - 2023.12)

Huawei Technologies Co., Ltd

- In short, [Compute Express Link](https://www.computeexpresslink.org/) is a processor-to-peripheral Cache-Coherent Interconnect. It is based on PCIe, but can do so much more. [This repo](https://github.com/twicy/awesome-CXL) collects basics, tutorial and news about CXL, I will try my best to update this regularly.
- Huawei happens to be a member of the CXL consortium, and I am lucky to be a part of the gigantic software-hardware codesign project of single node memory expansion using CXL 1.1 devices.
![](/images/projects/cxl.png)

### PEBS sampling V.S. Access bit checking --  (2022.12 - 2023.04)

Huawei Technologies Co., Ltd

- [etmem](https://github.com/openeuler-mirror/etmem), is Huawei's tiered memory system manager. It takes a traditional approach of access bit checking to determine the temperature of a page, the same spirit of [Nimble](https://dl.acm.org/doi/10.1145/3297858.3304024).
- But as many other researchers have pointed out, page table walking is time consuming and coarse-gained, while a CPU hardware event sampling method called PEBS sampling may be the cure. [HEMEM](https://dl.acm.org/doi/10.1145/3477132.3483550), [MEMTIS](https://dl.acm.org/doi/10.1145/3600006.3613167#:~:text=We%20present%20Memtis%2C%20a%20tiered,to%20the%20fast%20tier%20capacity), all adopt this method.
- However, is PEBS sampling really that powerful? Does it perform well under real world applications? Can PEBS sampling totally outperform page table scanning? My experience tells a slightly different story.
- Developed a substitute kernel module that listens to fine-grained (compared to access bit checking) hardware access counter results.
![](/images/projects/memory_tier.png)
image source: [TPP: Transparent Page Placement for CXL-Enabled Tiered-Memory](https://arxiv.org/abs/2206.02878)