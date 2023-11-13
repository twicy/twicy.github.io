---
layout: page
permalink: /projects/index.html
title: Projects
---

### OpenUCX Optimization

- [OpenUCX](https://openucx.org/) is an open-source, production-grade communication framework for data-centric and high-performance applications.
- It is based on a set of abstract communication primitives, inclusing RDMA. However, current implementation of openucx is not enough, as it fails to mitigate the widely known scalability issue of RDMA network.
- We have come up with several optimizations, with more service type choices and better communcation establishment methods and more.
![](/images/projects/openucx_layout.jpg)

### Memory node expansion utilizing FPGA-based CXL 1.1 devices

- In short, [Compute Express Link](https://www.computeexpresslink.org/) is a processor-to-peripheral Cache-Coherent Interconnect. It is based on PCIe, but can do so much more. [This repo](https://github.com/twicy/awesome-CXL) collects basics, tutorial and news about CXL, I will try my best to update this regularly.
- Huawei happens to be a member of the CXL consortium, and I am lucky to be a part of the gigantic software-hardware codesign project of single node memory expansion using CXL 1.1 devices.
![](/images/projects/cxl.png)

### PEBS sampling V.S. Access bit checking

- [etmem](https://github.com/openeuler-mirror/etmem), is Huawei's tiered memory system manager. It takes a traditional approach of access bit checking to determine the temperature of a page, the same spirit of [Nimble](https://dl.acm.org/doi/10.1145/3297858.3304024).
- But as many other researchers have pointed out, page table walking is time consuming and coarse-gained, while a CPU hardware event sampling method called PEBS sampling may be the cure. [HEMEM](https://dl.acm.org/doi/10.1145/3477132.3483550), [MEMTIS](https://dl.acm.org/doi/10.1145/3600006.3613167#:~:text=We%20present%20Memtis%2C%20a%20tiered,to%20the%20fast%20tier%20capacity), all adopt this method.
- However, is PEBS sampling really that powerful? Does it perform well under real world applications? Can PEBS sampling totally outperform page table scanning? My experience tells a slightly different story.
![](/images/projects/memory_tier.png)
image source: [TPP: Transparent Page Placement for CXL-Enabled Tiered-Memory](https://arxiv.org/abs/2206.02878)