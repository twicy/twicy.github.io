---
layout: page
permalink: /projects/index.html
title: Projects
---

This page is still under construction

### OpenUCX Optimization

### Memory node expansion utilizing FPGA-based CXL 1.1 devices

### PEBS sampling V.S. Access bit checking

- [etmem](https://github.com/openeuler-mirror/etmem), is Huawei's tiered memory system manager. It takes a traditional approach of access bit checking to determine the temperature of a page, the same spirit of [Nimble](https://dl.acm.org/doi/10.1145/3297858.3304024).
- But as many other researchers have pointed out, page table walking is time consuming and coarse-gained, while a CPU hardware event sampling method called PEBS sampling may be the cure. [HEMEM](https://dl.acm.org/doi/10.1145/3477132.3483550), [MEMTIS](https://dl.acm.org/doi/10.1145/3600006.3613167#:~:text=We%20present%20Memtis%2C%20a%20tiered,to%20the%20fast%20tier%20capacity), all adopt this method.
- However, is PEBS sampling really that powerful? Does it perform well under real world applications? My experience tells a whole different story.
