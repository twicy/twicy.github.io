# Introduction to CXL

## Three protocols

- CXL.io

The Home Agent discovers, enumerates, configures and manages CXL device through CXL.io

- CXL.mem

The Home Agent uses device memory the same way it is using CPU memory

- CXL.cache

Coherent cache of host and device

## Three Types of CXL Devices

all kinds of CXL Devices need to support CXL.io protocol

- Type 1: CXL.cache + CXL.io

PCIe may fail to support sophisticated `Atomic Operations`, to do so they will need system locks

CXL looks like a core's L2 cache

sample usage: PGAS NIC, NIC atomics

- Type 2: CXL.cache + CXL.io + CXL.mem

`Host-managed Device Memory`, namely `HDM`, different from `Private Device Memory` in PCIe

HDM-D and HDM-DB(CXL.mem Back-invalidation BISnp and BIRsp)

[block diagramL type 2 device or host accesses hdm-d region]

if CXL device is accessing main memory(Host BIOS), it looks like L2;

if CXL device is accessing `HDM-D` memory(CXL device memory), it is doing through another totally different diagram

[block diagramL type 2 device or host accesses hdm-db region]

sample usage: GPGPU, Dense Computation

- Type 3: CXL.io + CXL.mem

`HDM-H`

`core`=>`cache agent(L3)`=>`Home Agent`=>`Mem controller`=>`Main Memory`
`core`=>`cache agent(L3)`=>`Home Agent`=>`Dev Mem controller`=>`HDM-H memory`

sample usage: Memory BW expansion, Memory capacity expansion, Storage class memory

CXL_3.0 specification release_final
CXL_3.0_white paper final
www.intel.com/programmable/technical-pdfs/683852.pdf