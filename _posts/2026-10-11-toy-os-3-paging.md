---
layout: page
permalink: /blogs/os/toy-os-3-paging.html
title: "Toy OS 3: Protected Mode and Paging"
topic: Operating System and Kernel Bypassing
math: false
description: >-
  The third toy-os worklog follows the loader from real mode into 32-bit protected mode and
  then turns on paging. It enables A20, loads the GDT, sets CR0.PE and far-jumps to reload
  CS; Bochs shows the CPU in protected mode while CS still caches its real-mode descriptor.
  The loader then builds a page directory at 1 MiB that maps the first MiB twice (identity
  and at 0xC0000000) plus a recursive entry, moves the GDT, the video segment and the stack
  into the higher half, and sets CR0.PG. Maps show where the page tables sit in physical
  memory and what the virtual address space looks like afterwards.
---

# Toy OS 3: Protected Mode and Paging

*Toy OS worklog: [1. Real mode](/blogs/os/toy-os-1-real-mode.html) · [2. Segmentation](/blogs/os/toy-os-2-segmentation.html) · **3. Paging** · [4. Kernel](/blogs/os/toy-os-4-kernel.html) · [5. Interrupts](/blogs/os/toy-os-5-interrupts.html) · [6. Buddy](/blogs/os/toy-os-6-buddy.html) · [7. Slab](/blogs/os/toy-os-7-slab.html) · [Source](https://github.com/twicy/toy-os/tree/fc88a1de36e8cd2cf495c46f3e6b36c01fc94f8b)*

The loader at `0x800` has a GDT (post 2). It now does five things, in this order:

1. Ask the BIOS for the memory map (E820). BIOS calls only work in real mode, so this comes first ([post 6](/blogs/os/toy-os-6-buddy.html) covers it).
2. Switch to 32-bit protected mode.
3. Read `kernel.bin` from the disk into a buffer ([post 4](/blogs/os/toy-os-4-kernel.html)).
4. Build page tables and turn on paging, moving itself into the higher half at `0xC0000000`.
5. Copy the kernel into place and jump to it (post 4).

This post covers steps 2 and 4.

## Table of Contents

- [Into protected mode](#into-protected-mode)
- [Why paging](#why-paging)
- [The loader's page tables](#the-loaders-page-tables)
- [Turning paging on and moving to the higher half](#turning-paging-on-and-moving-to-the-higher-half)
- [The virtual address space now](#the-virtual-address-space-now)
- [The recursive entry](#the-recursive-entry)
- [References](#references)

## Into protected mode

```nasm
loader_entry:
    ; BIOS calls only work in real mode, collect the memory map first
    call detect_memory

    ; disable interrupts - no IDT yet, any interrupt in prot mode would triple fault
    cli

    ; enable A20
    in al, 0x92
    or al, 0000_0010b
    out 0x92, al

    ; load GDT
    lgdt [GDT_PTR]

    ; open protection mode - set cr0 bit 0
    mov eax, cr0
    or eax, 0x00000001
    mov cr0, eax

    ; refresh pipeline
    jmp dword SELECTOR_CODE:prot_mode_entry
```

- **`cli`**. In protected mode, interrupts go through an IDT, and there is none yet (post 5). The real-mode vector table at address 0 means nothing to a protected-mode CPU. The first timer tick would fault, the fault could not be delivered either (double fault), and then the CPU resets (triple fault). IF stays 0 until the kernel runs `sti` in post 5.
- **A20**. The 8086 wrapped addresses at 1 MiB, and some old software relied on it, so PCs keep address line 20 forced to 0 until someone enables it. With A20 off, every odd MiB is an alias of the even one below it: `0x100000` reads `0x000000`. Setting bit 1 of port `0x92` (the "fast A20" gate) turns it on.
- **`lgdt`, then `CR0.PE = 1`**.
- **The far jump**. Setting PE does not change how the CPU runs the next instruction. CS still holds the descriptor it cached in real mode, and instructions that were already fetched and decoded as 16-bit code are still in the pipeline. A far jump loads CS from GDT entry 1 and discards that prefetched code. `dword` makes it a 48-bit `selector:offset32` pointer: `66 EA 22 09 00 00 08 00`.

Bochs makes the in-between state visible. Here are the segment registers right *after* `mov cr0, eax` and *before* the jump, at `0x8A3`:

```text
(0) [0x0000000008a3] 0000:00000000000008a3 (unk. ctxt): jmpf 0x0008:00000922      ; 66ea220900000800
cs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
gdtr:base=0x0000000000000803, limit=0x1f
CR0=0x60000011: pg CD NW ac wp ne ET ts em mp PE
```

PE is set, yet CS is still the 64 KiB real-mode segment. One instruction later, at `prot_mode_entry`:

```text
(0) [0x000000000922] 0008:0000000000000922 (unk. ctxt): mov ax, 0x0010            ; 66b81000
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
```

The new code then reloads the other segment registers from the GDT and gets a stack:

```nasm
[bits 32]
prot_mode_entry:
    ; segment registers still hold real-mode values, reload with selectors
    mov ax, SELECTOR_DATA
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov esp, LOADER_STACK_TOP       ; 0x9F000
    mov ax, SELECTOR_VIDEO
    mov gs, ax
```

Without the BIOS, printing means writing character/attribute pairs straight into the text buffer through `gs`. The cursor position also has to be done by hand: it lives in two CRTC registers, index `0x0E` (high byte) and `0x0F` (low byte), behind ports `0x3D4`/`0x3D5`. `print_prot_mode` reads the position the BIOS left there, writes the string from that point, handles CR and LF, and writes the new position back, so the protected-mode lines continue right under the MBR's.

## Why paging

With flat segments (post 2), segmentation protects nothing. Paging gives every 4 KiB page its own mapping and its own permission bits (present, writable, user). That is what lets a kernel:

- live at the same virtual addresses in every process (Linux on 32-bit x86: user space `0`–`0xBFFFFFFF`, kernel `0xC0000000` and up),
- keep user code out of kernel memory, and
- use physical memory wherever it happens to be free.

32-bit x86 paging with 4 KiB pages uses two levels. CR3 holds the physical address of a *page directory* (1024 four-byte PDEs). Each PDE points to a *page table* (1024 PTEs), and each PTE points to a 4 KiB page frame. A virtual address is split into 10, 10 and 12 bits:

![VA 0xC00B8140 split into PD index 768, PT index 184 and offset 0x140, walked through the loader's page directory and PT0 to physical 0xB8140](/assets/images/posts/toy-os-3-paging/page-walk.svg)

*Translating VA `0xC00B8140`, the text buffer seen through the kernel alias, with the loader's real table contents.*

![PDE and PTE format: frame address in bits 31-12, then AVL, G, PS/PAT, D, A, PCD, PWT, U/S, R/W, P](/assets/images/posts/toy-os-3-paging/pte.svg)

*A PDE, `0x00101007`. The low 12 bits are flags, because tables and frames are 4 KiB aligned.*

## The loader's page tables

The loader puts the page directory at 1 MiB, the first address above the BIOS areas:

```nasm
PAGE_DIR_BASE equ 0x100000
PAGE_TABLE_0_BASE equ 0x101000
PDE_FLAGS equ ...       ; U/S | R/W | P = 0x7
PTE_FLAGS equ ...       ; same
PDE_0 equ PAGE_TABLE_0_BASE + PDE_FLAGS
PTE_0 equ PAGE_0_BASE + PTE_FLAGS
```

```nasm
setup_gpt:
    xor eax, eax
PURGE_LEN equ (1024 * 4 + 1024 * 1024 * 4)
.purge_mem:
    mov byte [PAGE_DIR_BASE + eax], 0
    inc eax
    cmp eax, PURGE_LEN
    jl .purge_mem

    ; Setting up PDE0 and PT0
    mov dword [PAGE_DIR_BASE], PDE_0
    mov dword [PAGE_DIR_BASE + 768 * 4], PDE_0

    xor eax, eax
    ; PTE for frame 0, advanced one frame per entry
    mov ebx, PTE_0

; Map the first 1M of memory
; VA [0x0, 0x100000) and VA[0xc0000000, 0xc0100000) map to PA [0x0, 0x100000)
.setup_pt0:
    mov dword [PAGE_TABLE_0_BASE + 4 * eax], ebx
    add ebx, 4096
    inc eax
    cmp eax, 256
    jl .setup_pt0

    ; Recursive Mapping of Page Directory
    mov dword [PAGE_DIR_BASE + 1023 * 4], PAGE_DIR_BASE + PDE_FLAGS

; Map other kernel PDEs
    mov ecx, 769
    mov ebx, PAGE_TABLE_0_BASE + 769 * 4096 + PDE_FLAGS
.setup_kernel_pdes:
    mov dword [PAGE_DIR_BASE + ecx * 4], ebx
    add ebx, 4096
    inc ecx
    cmp ecx, 1023
    jl .setup_kernel_pdes
    ret
```

The page directory gets four kinds of entries:

- **PDE 0 → PT0** and **PDE 768 → PT0**. PT0 maps frames 0–255, the first MiB. The same 1 MiB appears twice: at VA 0 (identity), so the loader keeps running after paging is on, and at VA `0xC0000000` (768 × 4 MiB), where the kernel will live.
- **PDE 769–1022 → their own page tables**, all of whose PTEs are zero. Nothing is mapped there yet, but every kernel PDE already exists. A future process's page directory can copy PDEs 768–1023 once, and the whole kernel half is shared from then on. Post 6 also relies on PDE 769 being there.
- **PDE 1023 → the page directory itself**, the recursive entry (below).

The page table for PDE *i* sits at `0x101000 + i × 4 KiB`. That needs no allocator, but it leaves slots 1–768 empty: 3 MiB of zeroed, never-used memory. Here is the physical side:

![Physical memory from 0 to 512 MiB: low 1 MiB, the page directory at 0x100000, PT0 at 0x101000, unused slots up to 0x402000, PTs 769-1022 up to 0x500000, an unused slot for 1023, then free RAM](/assets/images/posts/toy-os-3-paging/loader-pt-phys.svg)

*Physical memory taken by the loader's paging structures: `[0x100000, 0x501000)`, 4 MiB + 4 KiB. Post 6 gives it back.*

Zeroing that region byte by byte costs four instructions per byte. Bochs counts 7,348,009 instructions when `setup_gpt` is called and 24,144,543 at the magic breakpoint after paging is on. The difference, 16.8 million, is almost exactly 4 × `PURGE_LEN` (4,198,400 bytes). So two thirds of all instructions before the kernel starts go into this one loop. `rep stosd` would be four times fewer iterations. It does not matter in Bochs, but it is a fun number.

The flags are `0x7`: present, writable and **user**. With U/S = 1 everywhere, ring 3 code could read and write the kernel. That is harmless while nothing runs in ring 3, and the kernel's own page tables (post 6) drop the user bit.

## Turning paging on and moving to the higher half

```nasm
    call setup_gpt

; Move GDT, video segment and stack to their kernel aliases
    sgdt [GDT_PTR]
    ; video desc base 31..24 is the top byte of its high dword: 0xB8000 -> 0xC00B8000
    or dword [VIDEO_DESC + 4], 0xC0000000
    ; GDTR base 0x803 -> 0xC0000803
    add dword [GDT_PTR + 2], 0xC0000000
    add esp, 0xC0000000

; Enable paging
    mov eax, PAGE_DIR_BASE
    mov cr3, eax
    ; set cr0 bit 31 (PG)
    mov eax, cr0
    or eax, 0x80000000
    mov cr0, eax

    ; GDTR base is a linear address, load the high one only after paging is on
    lgdt [GDT_PTR]
    ; gs still caches the old video base until the selector is reloaded
    mov ax, SELECTOR_VIDEO
    mov gs, ax
```

The idea, as my commit note puts it: *mirror the stack pointer and video segment to kernel space after paging.* Everything the CPU finds by *linear* address has to move to its alias above `0xC0000000`, because the identity map will not last (post 6 removes it):

- **The GDTR base is a linear address.** After paging, the GDT is still reachable at `0x803` through the identity map, but the kernel should not depend on that. The new base, `0xC0000803`, is loaded with `lgdt` only once paging is on. Before that, `0xC0000803` would be a physical address, and there is no RAM there.
- **The video segment base**: `0xB8000` → `0xC00B8000`. Base bits 31..24 are the top byte of the descriptor's high dword, so a single `or` does it. `gs` must then be reloaded, or it keeps the base it cached earlier (post 2).
- **The stack**: `esp += 0xC0000000`. This happens *before* paging is on, when `0xC009F000` points at nothing. It works because nothing touches the stack between the `add` and `mov cr0, eax`: no `push`, no `call`.

Why does execution survive `mov cr0, eax` at all? The next instruction's address is still around `0x990`, and the identity map (PDE 0) translates it to the same physical address. Without PDE 0, the very next fetch would page-fault, with no IDT to handle it.

At the magic breakpoint (`xchg bx, bx`) right after this:

```text
gs:0x0018, dh=0xc0c0930b, dl=0x80000007, valid=7
	Data segment, base=0xc00b8000, limit=0x00007fff, Read/Write, Accessed
gdtr:base=0x00000000c0000803, limit=0x1f
CR0=0xe0000011: PG CD NW ac wp ne ET ts em mp PE
CR3=0x000000100000
```

![Bochs: Entering MBR, Successfully loaded loader, Successfully entered protection mode, Successfully enabled paging](/assets/images/posts/toy-os-3-paging/paging.png)

*The last line is written through `gs` with base `0xC00B8000`, so it went through the page tables.*

## The virtual address space now

`info tab` in Bochs lists every mapping in the current page tables:

```text
cr3: 0x000000100000
0x0000000000000000-0x00000000000fffff -> 0x000000000000-0x0000000fffff
0x00000000c0000000-0x00000000c00fffff -> 0x000000000000-0x0000000fffff
0x00000000ffc00000-0x00000000ffc00fff -> 0x000000101000-0x000000101fff
0x00000000fff00000-0x00000000fff00fff -> 0x000000101000-0x000000101fff
0x00000000fff01000-0x00000000ffffefff -> 0x000000402000-0x0000004fffff
0x00000000fffff000-0x00000000ffffffff -> 0x000000100000-0x000000100fff
```

![Virtual address space after paging: identity map of 1 MiB, the kernel alias at 0xC0000000, empty kernel PDEs, and the page tables at 0xFFC00000; physical: low 1 MiB, PD+PTs, rest of RAM unmapped](/assets/images/posts/toy-os-3-paging/loader-va.svg)

*The loader's virtual address space (left) and where it lands in physical memory (right). Hatched = no mapping; touching it is a page fault.*

The first two lines are the double mapping of the first MiB. The other four all come from one entry, PDE 1023.

## The recursive entry

PDE 1023 points at the page directory itself. A PDE and a PTE have the same format, so the CPU happily uses the directory as if it were a page table. For any VA in the top 4 MiB, `0xFFC00000`–`0xFFFFFFFF`:

1. The PD index is 1023, so the "page table" is the page directory.
2. The PT index *i* picks PDE *i*, so the "page frame" is page table *i*.

So the top 4 MiB is a window onto all the page tables, laid out in PDE order, and the last page of it, PT index 1023, is the directory itself. The comment in `loader.S` works through VA `0xFFC00001`: PD index 1023 → the PD; PT index 0 → PDE 0 → PT0 at `0x101000`; offset 1 → byte 1 of PT0.

![The recursive window from 0xFFC00000 to 4 GiB: PT0 twice, PTs 769-1022, the page directory at 0xFFFFF000, and holes for empty PDEs](/assets/images/posts/toy-os-3-paging/recursive.svg)

*The top 4 MiB through PDE 1023. It matches the last four lines of `info tab`.*

As my commit note says, this trick only works because PDEs and PTEs share the same format. It lets code edit a page table *by virtual address* without knowing where the table sits physically. Two formulas give the entry for any VA:

```c
pte_t *pte = (pte_t *)(0xFFC00000 + (va >> 12) * 4);
pde_t *pde = (pde_t *)(0xFFFFF000 + (va >> 22) * 4);
```

The kernel uses the first one exactly once, in post 6, to map the frames of its new page tables before it switches to them.

[Post 4](/blogs/os/toy-os-4-kernel.html) is the last thing the loader does: copy the kernel's ELF segments to `0xC0001000` and jump to `main`.

## References

1. 郑钢, 《操作系统真相还原》, ch. 4 (进入保护模式: A20, CR0.PE, flushing the pipeline) and ch. 5 (启用内存分页机制: page directory at 1 MiB, the kernel at 3 GiB, PDE 1023 pointing at the directory).
2. navi, [从零开始写 OS 内核 - GDT 与保护模式](https://segmentfault.com/a/1190000040153113) and [虚拟内存初探](https://segmentfault.com/a/1190000040164669), segmentfault.
3. D. P. Bovet and M. Cesati, *Understanding the Linux Kernel*, 3rd ed., ch. 2 "Memory Addressing": "Paging in Hardware" (regular paging) and "Paging in Linux" (the kernel's 3 GiB / 1 GiB split).
4. Intel 64 and IA-32 Architectures Software Developer's Manual, Vol. 3A, "Paging": 32-bit paging, PDE and PTE formats.
5. OSDev Wiki, [A20 Line](https://wiki.osdev.org/A20_Line).
