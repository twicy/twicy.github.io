---
layout: page
permalink: /blogs/os/toy-os-2-segmentation.html
title: "Toy OS 2: Segmentation and the GDT"
topic: Operating System and Kernel Bypassing
math: false
description: >-
  The second toy-os worklog covers x86 segmentation, which has to be set up before the
  switch to protected mode. In protected mode a segment register holds a selector, and the
  CPU caches the base, limit and access rights of the descriptor it names in the Global
  Descriptor Table. The post decodes a descriptor bit by bit, traces one address through the
  loader's four-entry GDT, and explains why toy-os, like Linux, uses flat 4 GiB segments.
  Once in C, the kernel replaces that GDT with its own kernel and user segments, which end
  up in the last 128 bytes of where the MBR used to be.
---

# Toy OS 2: Segmentation and the GDT

*Toy OS worklog: [1. Real mode](/blogs/os/toy-os-1-real-mode.html) · **2. Segmentation** · [3. Paging](/blogs/os/toy-os-3-paging.html) · [4. Kernel](/blogs/os/toy-os-4-kernel.html) · [5. Interrupts](/blogs/os/toy-os-5-interrupts.html) · [6. Buddy](/blogs/os/toy-os-6-buddy.html) · [7. Slab](/blogs/os/toy-os-7-slab.html) · [Source](https://github.com/twicy/toy-os/tree/fc88a1de36e8cd2cf495c46f3e6b36c01fc94f8b)*

The loader at `0x800` (post 1) wants to leave real mode. In protected mode a segment register no longer holds `address / 16`. It holds a *selector*, an index into a table of *descriptors* that the OS writes in memory. So before setting `CR0.PE`, the loader needs that table: the Global Descriptor Table (GDT). This post covers the GDT. [Post 3](/blogs/os/toy-os-3-paging.html) does the actual switch.

## Table of Contents

- [Visible and hidden parts of a segment register](#visible-and-hidden-parts-of-a-segment-register)
- [Selectors](#selectors)
- [Descriptors](#descriptors)
- [The loader's GDT](#the-loaders-gdt)
- [Flat segments, like Linux](#flat-segments-like-linux)
- [The kernel's own GDT](#the-kernels-own-gdt)
- [References](#references)

## Visible and hidden parts of a segment register

Each segment register (`cs`, `ds`, `es`, `fs`, `gs`, `ss`) has a 16-bit visible part, the selector, and a hidden part, the descriptor cache: base, limit and access rights. When a selector is loaded (`mov ds, ax`, or a far jump for `cs`), the CPU reads the descriptor from the table, checks it, and copies it into the cache. Every later memory access uses the cache, not the table.

Bochs's `sreg` prints both. In the line below, `dh` and `dl` are the two dwords of the cached descriptor:

```text
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
```

Post 1 already showed the hidden part at work: at reset, CS has selector `0xF000` but base `0xFFFF0000`. In real mode a selector load sets base = selector × 16. In protected mode the base comes from the descriptor.

The CPU turns a *logical* address (selector:offset) into a *linear* address in three steps:

1. Find the descriptor: index × 8 into the table that GDTR points at.
2. Check the offset against the limit, and the access against the type and privilege. A failed check raises #GP.
3. Linear address = base + offset. Paging is still off, so linear = physical.

Here is one real access. The loader's `print_prot_mode` writes `mov [gs:ebx * 2], ax`. For the first character of row 2 of the screen, `ebx * 2 = 0x140`:

![gs = 0x18 selects GDT entry 3, the video segment with base 0xB8000, so offset 0x140 becomes linear address 0xB8140](/assets/images/posts/toy-os-2-segmentation/translation.svg)

*A logical address through the loader's GDT. The CPU actually reads the descriptor once, when `gs` is loaded, not on every access.*

## Selectors

![A selector: bits 15-3 are the descriptor index, bit 2 the table indicator, bits 1-0 the RPL](/assets/images/posts/toy-os-2-segmentation/selector.svg)

*A selector. With TI = 0 and RPL = 0 it is simply the byte offset of the descriptor in the GDT.*

`boot.inc` builds the three selectors the loader uses:

```nasm
SELECTOR_CODE equ (1 << SELECTOR_DESC_IDX_OFFSET) + \
                  (0b << SELECTOR_TI_OFFSET) + \
                  (00b << SELECTOR_RPL_OFFSET)
SELECTOR_DATA equ (2 << SELECTOR_DESC_IDX_OFFSET) + ...
SELECTOR_VIDEO equ (3 << SELECTOR_DESC_IDX_OFFSET) + ...
```

Index 0 must be the null descriptor. Loading selector 0 into `ds` is allowed (it marks the register unusable), but any access through it faults.

## Descriptors

A descriptor is 8 bytes. The layout is odd because of history: the 80286 had 24-bit bases and 16-bit limits, and the 386 squeezed the missing high bits into the word the 286 had left reserved.

![The loader's code descriptor bit by bit: base 0, limit 0xFFFFF, G=1, D/B=1, P=1, DPL=0, S=1, type 1010](/assets/images/posts/toy-os-2-segmentation/descriptor.svg)

*The loader's code descriptor, `0x00CF9A00_0000FFFF`.*

- **Base** (32 bits, in three pieces) and **limit** (20 bits, in two). With **G = 1** the limit counts 4 KiB units, so `0xFFFFF` means the whole 4 GiB.
- **D/B = 1**: a 32-bit segment (default operand size and `esp` for stacks).
- **P = 1**: present. **DPL**: the privilege level the descriptor belongs to (0 = kernel, 3 = user).
- **S = 1**: a code or data segment. With S = 0 it would be a system descriptor, such as a TSS or a gate.
- **Type** for code is X C R A: executable, not conforming, readable, accessed = `1010`. For data it is X E W A: not executable, expands up, writable = `0010`. The CPU sets A when it loads the descriptor, which is why Bochs shows `dh=0x00cf9b00` (type `1011`).

Instead of a magic `0x00CF9A00`, `boot.inc` composes each field with a shift:

```nasm
; -------- Code segment --------
; Base: 0, Limit: 0xFFFFF * 4K = 4G
; Type: x=1,c=0,r=1,a=0
DESC_CODE_HIGH_32 equ (0 << DESC_BASE_31_24_OFFSET) + \
                      (1 << DESC_G_OFFSET) + \
                      (1 << DESC_DB_OFFSET) + \
                      (0 << DESC_L_OFFSET) + \
                      (0 << DESC_AVL_OFFSET) + \
                      (1111b << DESC_LIMIT_19_16_OFFSET) + \
                      (1 << DESC_P_OFFSET) + \
                      (00b << DESC_DPL_OFFSET) + \
                      (1 << DESC_S_OFFSET) + \
                      (1010b << DESC_TYPE_OFFSET) + \
                      (0 << DESC_BASE_23_16_OFFSET)

DESC_CODE_LOW_32 equ (0 << DESC_BASE_15_0_OFFSET) + \
                     (0xFFFF << DESC_LIMIT_15_0_OFFSET)
```

## The loader's GDT

The table lives inside `loader.bin`, right after its first instruction:

```nasm
section loader vstart=LOADER_BASE_ADDR
jmp loader_entry

; temporary gdt to enter protection mode
GDT_BASE:
RESV_DESC:
    dd 0x0
    dd 0x0
CODE_DESC:
    dd DESC_CODE_LOW_32
    dd DESC_CODE_HIGH_32
DATA_DESC:
    dd DESC_DATA_LOW_32
    dd DESC_DATA_HIGH_32
VIDEO_DESC:
    dd DESC_VIDEO_LOW_32
    dd DESC_VIDEO_HIGH_32
GDT_SIZE equ $ - GDT_BASE
GDT_LIMIT equ GDT_SIZE - 1

; lgdt 48-bit
GDT_PTR:
    dw GDT_LIMIT
    dd GDT_BASE
```

`jmp loader_entry` assembles to 3 bytes (`E9 87 00`), so in memory:

| Physical | Bytes | What |
| --- | --- | --- |
| `0x800`–`0x802` | 3 | `jmp loader_entry` |
| `0x803`–`0x822` | 32 | GDT: null, code, data, video |
| `0x823`–`0x828` | 6 | GDTR operand for `lgdt`: limit `0x001F`, base `0x00000803` |
| `0x829`– | | messages, then code |

`lgdt [GDT_PTR]` copies those 6 bytes into GDTR. The table itself stays where it is: the CPU reads it from memory every time a selector is loaded. At the first protected-mode instruction (`0x922`, post 3), Bochs shows:

```text
gdtr:base=0x0000000000000803, limit=0x1f
gdt (base=0x0000000000000803, limit=31):
gdt[0x0000]=<null entry>
gdt[0x0008]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
gdt[0x0010]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write
gdt[0x0018]=Data segment, base=0x000b8000, limit=0x00007fff, Read/Write
```

The video descriptor is the only interesting one. Its base is `0xB8000` and its limit is `0x0007` with G = 1, so the limit is (7 + 1) × 4 KiB − 1 = `0x7FFF`: 32 KiB, exactly the color text buffer `0xB8000`–`0xBFFFF`. A write through `gs` past the buffer raises #GP instead of scribbling over the VGA BIOS ROM. That limit check is the only real protection segmentation gives toy-os.

![The linear address space: the code and data segments cover all 4 GiB, the video segment only 0xB8000 to 0xBFFFF](/assets/images/posts/toy-os-2-segmentation/segments.svg)

*What each of the loader's segments can reach.*

## Flat segments, like Linux

Code and data are *flat*: base 0, limit 4 GiB. Translation does nothing, offset = linear address, and a C pointer is just a linear address. ULK (ch. 2, "Segmentation in Linux") describes Linux doing the same. Its four main segments, `__KERNEL_CS`, `__KERNEL_DS`, `__USER_CS` and `__USER_DS`, all have base 0 and limit 2<sup>32</sup> − 1. They differ only in type and DPL. ULK's reasons are that memory management is simpler when every process uses the same linear addresses, and that RISC architectures barely support segmentation. Protection moves to paging (post 3). Segments are left with one job: their DPL decides whether the CPU runs at ring 0 or ring 3.

## The kernel's own GDT

The loader's GDT has two problems. It lives inside `loader.bin`, which is dead once the kernel runs, and it has no user segments. In the revision commit ([`cbd7402`](https://github.com/twicy/toy-os/commit/cbd7402)), `main()` starts with `init_gdt()`, which builds a new table in the kernel's `.bss`:

```c
/* GDT indices, a selector is index << 3 | RPL
 * kernel code must stay at 1: the loader's cs and every IDT gate use selector 0x08 */
#define KERNEL_CODE_IDX 0x1
#define KERNEL_DATA_IDX 0x2
#define USER_CODE_IDX 0x3
#define USER_DATA_IDX 0x4

struct gdt_entry {
    uint16_t limit_low;
    uint16_t base_low;
    uint8_t base_middle;
    uint8_t access;
    uint8_t granularity;
    uint8_t base_high;
} __attribute__((packed));
```

```c
static struct gdt_entry gdt[MAX_GDT_ENTRIES];
static struct gdtr gdtr;

void init_gdt(void) {
    gdt[0] = (struct gdt_entry){0};

    gdt[KERNEL_CODE_IDX].limit_low = 0xFFFF;
    gdt[KERNEL_CODE_IDX].access = 0x9A; // Present, Ring 0, Code Segment
    gdt[KERNEL_CODE_IDX].granularity = 0xCF; // 4K granularity, 32-bit
    ...
    gdtr.limit = sizeof(gdt) - 1;
    gdtr.base = (uint32_t)&gdt;

    // lgdt only changes where descriptors are read from, each segment register keeps
    // the descriptor it cached when loaded, so reload them all from the new table
    // cs can only be reloaded by a far jump
    asm volatile("lgdt %0\n\t"
                 "ljmp %1, $1f\n"
                 "1:\n\t"
                 "mov %2, %%ax\n\t"
                 "mov %%ax, %%ds\n\t"
                 "mov %%ax, %%es\n\t"
                 "mov %%ax, %%fs\n\t"
                 "mov %%ax, %%gs\n\t"
                 "mov %%ax, %%ss"
                 : : "m"(gdtr), "i"(KERNEL_CODE_SELECTOR), "i"(KERNEL_DATA_SELECTOR)
                 : "eax", "memory");
}
```

| Index | Selector | `access` | Bits P · DPL · S · type | Segment |
| --- | --- | --- | --- | --- |
| 1 | `0x08` | `0x9A` | 1 · 00 · 1 · 1010 | kernel code |
| 2 | `0x10` | `0x92` | 1 · 00 · 1 · 0010 | kernel data |
| 3 | `0x18` (`0x1B` with RPL 3) | `0xFA` | 1 · 11 · 1 · 1010 | user code |
| 4 | `0x20` (`0x23` with RPL 3) | `0xF2` | 1 · 11 · 1 · 0010 | user data |

`granularity = 0xCF` is the high nibble G = 1, D/B = 1 and the low nibble limit bits 19:16 = `0xF`. All four segments are flat, so they differ only in DPL and type, as in Linux. Index 3 used to be the video segment. The kernel's VGA code now writes to `0xC00B8000` through the flat data segment, so `gs` becomes plain `0x10`.

The `lgdt` comment is the important part. GDTR only says where descriptors are *read from*, and every segment register keeps the descriptor it cached when it was loaded. If `cs` and `ss` were not reloaded, they would keep pointing into the loader's table, and that would only show up much later, as a fault the first time something reloads them. `cs` cannot be the target of a `mov`, hence the `ljmp` to the next line. Bochs right after `init_gdt()`:

```text
gdt (base=0x00000000c0007d80, limit=127):
gdt[0x0000]=<null entry>
gdt[0x0008]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
gdt[0x0010]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gdt[0x0018]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, 32-bit
gdt[0x0020]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write
gdt[0x0028]=<null entry>
...
gs:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
```

The table has 16 entries (`limit = 16 × 8 − 1 = 127`). The free ones are for later, such as a TSS. It sits at VA `0xC0007D80`, which is physical `0x7D80`–`0x7DFF`, the last 128 bytes of where the MBR was loaded in post 1. The kernel image has long since overwritten the MBR (post 4).

A word on privilege, which post 5 builds on. The low two bits of `cs` are the *current privilege level* (CPL). A selector carries a *requested* privilege level (RPL), and a descriptor its DPL. To load a data segment, the CPU requires max(CPL, RPL) ≤ DPL. Everything in toy-os runs at CPL 0 so far, so the checks always pass.

## References

1. 郑钢, 《操作系统真相还原》, ch. 4 (保护模式入门): segment descriptors, the GDT, selectors.
2. navi, [从零开始写 OS 内核 - GDT 与保护模式](https://segmentfault.com/a/1190000040153113) and [全局描述符表 GDT](https://segmentfault.com/a/1190000040187304), segmentfault.
3. D. P. Bovet and M. Cesati, *Understanding the Linux Kernel*, 3rd ed., ch. 2 "Memory Addressing": "Segmentation in Hardware" and "Segmentation in Linux".
4. Intel 64 and IA-32 Architectures Software Developer's Manual, Vol. 3A, "Protected-Mode Memory Management": segment selectors, segment descriptors, descriptor tables.
