---
layout: page
permalink: /blogs/os/toy-os-6-buddy.html
title: "Toy OS 6: Physical Memory, E820 and the Buddy System"
topic: Operating System and Kernel Bypassing
math: false
description: >-
  The sixth toy-os worklog takes over the rest of RAM. The loader asks the BIOS for the
  E820 memory map, which in Bochs has five entries. A memblock-style boot allocator keeps
  sorted usable and reserved lists and places the kernel's new page tables, which map all
  512 MiB at 0xC0000000. They are mapped first through the loader's recursive page-table
  window. The loader's 4 MiB of page tables is then freed, and the struct page array lands
  exactly there. Every free frame goes to a buddy allocator with orders 0 to 10, which
  finds buddies by flipping one bit of the page frame number. Maps and the free lists, read
  out of Bochs, show where all of it ends up.
---

# Toy OS 6: Physical Memory, E820 and the Buddy System

*Toy OS worklog: [1. Real mode](/blogs/os/toy-os-1-real-mode.html) · [2. Segmentation](/blogs/os/toy-os-2-segmentation.html) · [3. Paging](/blogs/os/toy-os-3-paging.html) · [4. Kernel](/blogs/os/toy-os-4-kernel.html) · [5. Interrupts](/blogs/os/toy-os-5-interrupts.html) · **6. Buddy** · [7. Slab](/blogs/os/toy-os-7-slab.html) · [Source](https://github.com/twicy/toy-os/tree/fc88a1de36e8cd2cf495c46f3e6b36c01fc94f8b)*

So far the kernel lives inside the first MiB, and the only other memory it knows about is the loader's page tables. Bochs gives it 512 MiB. To hand out page frames, the kernel needs four things, in this order:

1. **What RAM exists**: the BIOS's E820 map, collected by the loader in real mode.
2. **A boot-time allocator** to place its own data structures before any real allocator exists (memblock).
3. **Page tables that map all of RAM**, so that any frame can be reached through a pointer.
4. **A `struct page` for every frame**, and the buddy allocator on top of them.

That is the order of `main()` (with the work spread over commits [`cbd7402`](https://github.com/twicy/toy-os/commit/cbd7402), [`8625d4f`](https://github.com/twicy/toy-os/commit/8625d4f), [`7a8ec84`](https://github.com/twicy/toy-os/commit/7a8ec84), [`cd46473`](https://github.com/twicy/toy-os/commit/cd46473) and [`ab8312c`](https://github.com/twicy/toy-os/commit/ab8312c)):

```c
    /* GDT's memory is safely pre-allcoated in BSS section, under _end */
    /* But kernel PG and PTs are phys_mem_alloc()ed already */
    e820_probe_phys_mem();
    init_pd();
    // Hand over physical pages to zone
    init_page_alloc();
```

The design follows Linux as ULK describes it (ch. 8, "Page Frame Management"), cut down to one zone.

## Table of Contents

- [E820: asking the BIOS](#e820-asking-the-bios)
- [memblock: a boot-time allocator](#memblock-a-boot-time-allocator)
- [Kernel page tables for all of RAM](#kernel-page-tables-for-all-of-ram)
- [struct page and mem_map](#struct-page-and-mem_map)
- [The buddy system](#the-buddy-system)
- [Physical memory after init_page_alloc()](#physical-memory-after-init_page_alloc)
- [References](#references)

## E820: asking the BIOS

`int 0x15` with `EAX = 0xE820` returns the memory map one entry per call. The caller passes `EDX = 'SMAP'`, an entry size in `ECX`, a buffer in `ES:DI`, and a continuation value in `EBX` (0 the first time). The BIOS hands back the next `EBX`, and `EBX = 0` means that was the last entry. Each entry is a 64-bit base, a 64-bit length and a 32-bit type (1 usable, 2 reserved, 3 ACPI reclaimable, 4 ACPI NVS, 5 bad). This is a BIOS call, so the loader does it in real mode, before anything else (post 3):

```nasm
; Store the BIOS E820 memory map at MEM_MAP_ADDR (layout in boot.inc)
detect_memory:
    mov dword [MEM_MAP_COUNT_ADDR], 0
    ; es:di = where the BIOS writes the next entry
    mov di, MEM_MAP_ENTRIES_ADDR
    ; continuation value, 0 = start from the first entry
    xor ebx, ebx
.next_entry:
    mov eax, 0xe820
    ; 'SMAP', set on every call since some BIOSes trash edx
    mov edx, 0x534d4150
    mov ecx, MEM_MAP_ENTRY_SIZE
    int 0x15
    ; cf set: no E820 on the first call, end of list on later ones
    jc .done
    ; eax = 'SMAP' confirms the BIOS understood the call
    cmp eax, 0x534d4150
    jne .done
    ; skip entries with a 0 qword length at offset 8
    mov eax, [di + 8]
    or eax, [di + 12]
    jz .skip
    inc dword [MEM_MAP_COUNT_ADDR]
    add di, MEM_MAP_ENTRY_SIZE
    ; stop before the map grows into the loader's stack
    cmp dword [MEM_MAP_COUNT_ADDR], MEM_MAP_MAX_ENTRIES
    jae .done
.skip:
    ; ebx = 0: that was the last entry
    test ebx, ebx
    jnz .next_entry
```

The map goes to physical `0x500`: a dword count, then up to 32 entries of 20 bytes, ending at `0x784`, just below the loader at `0x800` (post 1). Dumping it in Bochs right after the call:

```text
<bochs> xp /28wx 0x500
0x0000000000000500 <bogus+       0>:	0x00000005	0x00000000	0x00000000	0x0009f000
0x0000000000000510 <bogus+      16>:	0x00000000	0x00000001	0x0009f000	0x00000000
0x0000000000000520 <bogus+      32>:	0x00001000	0x00000000	0x00000002	0x000e8000
0x0000000000000530 <bogus+      48>:	0x00000000	0x00018000	0x00000000	0x00000002
0x0000000000000540 <bogus+      64>:	0x00100000	0x00000000	0x1ff00000	0x00000000
0x0000000000000550 <bogus+      80>:	0x00000001	0xfffc0000	0x00000000	0x00040000
0x0000000000000560 <bogus+      96>:	0x00000000	0x00000002	0x00000000	0x00000000
```

| # | Base | Length | End | Type |
| --- | --- | --- | --- | --- |
| 0 | `0x00000000` | `0x0009F000` | `0x0009F000` | 1, usable (636 KiB) |
| 1 | `0x0009F000` | `0x00001000` | `0x000A0000` | 2, reserved (EBDA) |
| 2 | `0x000E8000` | `0x00018000` | `0x00100000` | 2, reserved (BIOS) |
| 3 | `0x00100000` | `0x1FF00000` | `0x20000000` | 1, usable (511 MiB) |
| 4 | `0xFFFC0000` | `0x00040000` | `0x1_00000000` | 2, reserved (BIOS ROM at the top of 4 GiB) |

![The physical address space by E820: usable RAM below 0x9F000 and from 1 MiB to 512 MiB, reserved BIOS areas, and large ranges that are not listed at all](/assets/images/posts/toy-os-6-buddy/e820-space.svg)

*All 4 GiB of physical address space according to E820 (not to scale).*

Two things to notice. First, the map only lists ranges, so holes such as `[0xA0000, 0xE8000)` (video memory, VGA ROM) are simply absent, and so is everything between 512 MiB and the BIOS ROM. Second, entry 4 *ends* at `0x1_0000_0000`, which wraps to 0 in 32 bits. `e820_probe_phys_mem` therefore computes in 64 bits and clips to the last page boundary that fits in a `uint32_t`:

```c
        // 64 bit: the BIOS area entry 0xfffc0000 + 0x40000 = 0x100000000 wraps to 0 in 32 bits
        uint64_t start64 = ((uint64_t)entry->base_addr_high << 32) | entry->base_addr_low;
        uint64_t end64 = start64 + (((uint64_t)entry->length_high << 32) | entry->length_low);
        ...
        if (entry->type != E820_ARDS_TYPE_Memory || start64 >= PHYS_ADDR_LIMIT) {
            continue;
        }
        ...
        uint32_t entry_start = ALIGN_UP((uint32_t)start64, PAGE_SIZE);
        uint32_t entry_end = ALIGN_DOWN((uint32_t)end64, PAGE_SIZE);

        block_list_add(&ards_mem, entry_start, entry_end);
```

Only type 1 ranges are kept, rounded *inward* to whole pages, since a partial page cannot be handed out.

## memblock: a boot-time allocator

The page tables and `mem_map` need memory before the page allocator exists. Linux solves this with *memblock* (older kernels used a bitmap-based bootmem allocator): two sorted lists of physical ranges, *memory* and *reserved*. A boot-time allocation is a first fit in "memory minus reserved", and allocating just means adding a reserved range. toy-os does the same with two fixed arrays:

```c
/* ascending and non-overlapping */
struct phys_mem_list {
    struct phys_mem_block blocks[PHYS_MEM_MAX_BLOCKS];
    uint32_t count;
};

/* usable RAM from E820, filled once by e820_probe_phys_mem and never changed after */
static struct phys_mem_list ards_mem;
/* free mem = ards_mem - ards_rsved, will only be used once */
static struct phys_mem_list ards_rsved;
```

- `block_list_add` inserts `[start, end)` and merges it with every block it overlaps or touches.
- `block_list_remove` trims a block's head or tail, deletes a fully covered block, or splits a block in two.
- `phys_mem_alloc(size, align)` tries aligned candidates in each usable block. When a candidate overlaps a reserved block, it jumps past that block and tries again.
- `phys_mem_for_each_free(fn)` walks both sorted lists in step and calls `fn` on each free gap. That is how everything ends up in the buddy allocator below.

Right after probing, two ranges are reserved. The first is everything below `max(1 MiB, _end)`: the kernel, the BIOS areas, the dead loader and buffer, the stack. The second is the loader's page tables at `[0x100000, 0x501000)` (post 3). They touch, so the list merges them into one block. Commit [`8625d4f`](https://github.com/twicy/toy-os/commit/8625d4f) printed the lists:

![Bochs: usable memory 0x0-0x9F000 and 0x100000-0x20000000, reserved 0x0-0x501000, max_pfn 0x20000](/assets/images/posts/toy-os-6-buddy/memblock.png)

*`dump_phys_mem()` at commit [`8625d4f`](https://github.com/twicy/toy-os/commit/8625d4f). The 5 E820 entries became 2 usable blocks.*

`max_pfn = 0x20000` is one past the highest usable page frame number: 131072 frames of 4 KiB = 512 MiB.

## Kernel page tables for all of RAM

The loader's tables map only the first MiB. To reach any frame through a C pointer, the kernel maps *all* RAM at `0xC0000000`, the way Linux's direct mapping ("low memory") does. For 512 MiB that takes 128 page tables plus one page directory:

```c
void init_pd(void) {
    uint32_t n_pts = DIV_ROUND_UP(max_pfn, PT_ENTRIES);
    uint32_t first_pde = PD_INDEX(KERNEL_OFFSET);

    if (first_pde + n_pts > PD_ENTRIES) {
        PANIC("%u MiB of RAM does not fit the kernel's 1 GiB of address space",
            max_pfn >> (20 - PAGE_SHIFT));
    }

    /* Create a new kernel page dir, not colliding with loader's */
    kernel_pd = early_alloc(PAGE_SIZE);
    kernel_pts = early_alloc(n_pts * PAGE_SIZE);
    memset(kernel_pd, 0, PAGE_SIZE);
    memset(kernel_pts, 0, n_pts * PAGE_SIZE);

    for (uint32_t pfn = 0; pfn < max_pfn; pfn++) {
        kernel_pts[pfn] = mk_pte(pfn << PAGE_SHIFT, PG_KERNEL);
    }
    for (uint32_t i = 0; i < n_pts; i++) {
        kernel_pd[first_pde + i] = mk_pde(virt_to_phys(&kernel_pts[i * PT_ENTRIES]), PG_KERNEL);
    }

    /* no identity map of low memory, so virtual 0 faults, and no recursive entry:
     * page tables are reached through the direct map from here on */
    asm volatile ("mov %0, %%cr3" : : "r"(virt_to_phys(kernel_pd)) : "memory");

    /* the CPU no longer walks the loader's page directory and page tables, give them back */
    phys_mem_free(LOADER_PAGE_TABLES_START, LOADER_PAGE_TABLES_END);
}
```

There is a chicken-and-egg problem. `phys_mem_alloc` returns `0x501000`, the first free byte, but the CPU still walks the loader's tables, which map nothing above 1 MiB. So `memset(kernel_pd, ...)` at `0xC0501000` would page-fault. `early_alloc` maps each new frame first, by writing a PTE into the loader's tables through the recursive window from post 3:

```c
/* A brief recap of loader pt mapping:
 * PD starts at 0x100000, PTs start at 0x101000
 * PDE 0 and both PDE 768 point to PT0, which covers the first 1 MB
 * PDE 769 - 1022 are mapped to PT 769 - 1022, but PTEs are empty
 * PDE 1023 points to PD
 */
#define LOADER_PTES ((pte_t *)0xFFC00000)

/* Mapping kernel_pd and kernel_pts virtual addr, for later arr + index access */
static void *early_alloc(uint32_t size) {
    uint32_t phys = phys_mem_alloc(size, PAGE_SIZE);
    ...
    for (uint32_t off = 0; off < size; off += PAGE_SIZE) {
        uint32_t va = (uint32_t)phys_to_virt(phys + off);
        /* All PTs (arry of PTEs) are packed */
        LOADER_PTES[va >> PAGE_SHIFT] = mk_pte(phys + off, PG_KERNEL);
        asm volatile ("invlpg (%0)" : : "r"(va) : "memory");
    }
    return phys_to_virt(phys);
}
```

VA `0xC0501000` falls under PDE 769, and the loader created PT 769 in advance (empty), exactly for cases like this. As my commit note says, adding the new tables to the loader's tables makes their virtual addresses valid, which saves working with physical addresses. Two details:

- `kernel_pts` is one contiguous 512 KiB array, so the PTE for frame `pfn` is simply `kernel_pts[pfn]`.
- The flags are `PG_KERNEL`: present and writable, without the user bit the loader used.

After `mov cr3`, Bochs sees a single mapping:

```text
cr3: 0x000000501000
0x00000000c0000000-0x00000000dfffffff -> 0x000000000000-0x00001fffffff
```

![The kernel's virtual address space: nothing below 0xC0000000, all 512 MiB of RAM at 0xC0000000-0xDFFFFFFF, nothing above](/assets/images/posts/toy-os-6-buddy/kernel-va.svg)

*The kernel page directory: one direct map, nothing else. Compare with the loader's six mappings in post 3.*

The identity map is gone, so a NULL pointer dereference now page-faults, and the fault handler from post 5 names it. The recursive entry is gone too: page tables are now ordinary memory in the direct map. The `PANIC` in `init_pd` marks this design's limit. With more than 1 GiB of RAM, the direct map would not fit in the kernel's 1 GiB of virtual space. Linux stops its direct map at 896 MiB to keep 128 MiB free for vmalloc and temporary mappings, and treats the rest as *high memory* (ULK ch. 8). toy-os has no high memory.

## struct page and mem_map

The buddy allocator needs a descriptor for every page frame: `struct page`, in one array called `mem_map`, indexed by page frame number (pfn):

```c
struct page {
    uint16_t flags;
    uint16_t order;
    int32_t refcount;
    /* buddy: the free list of its order, slab: the partial list of its cache */
    struct list_head lru;
    /* PG_slab only, a slab is exactly one page so these describe the whole slab */
    struct kmem_cache *slab_cache;
    void *freelist;                 /* first free object, NULL when the slab is full */
    uint32_t inuse;                 /* objects handed out, 0 is empty, cache->objects is full */
};

extern struct page *mem_map;
#define pfn_to_page(pfn) (mem_map + (pfn))
#define page_to_pfn(page) ((uint32_t)((page) - mem_map))
#define page_address(page) phys_to_virt(page_to_phys(page))
#define virt_to_page(va) pfn_to_page(virt_to_phys(va) >> PAGE_SHIFT)
```

In the buddy commit it was 16 bytes: `flags`, `order`, `refcount` and `lru`. The slab fields arrive in [post 7](/blogs/os/toy-os-7-slab.html), which makes it 28 bytes, and 131072 × 28 B = 3.5 MiB:

```c
void init_page_alloc(void) {
    uint32_t size = max_pfn * sizeof(struct page);
    uint32_t phys = phys_mem_alloc(size, PAGE_SIZE);
    ...
    mem_map = phys_to_virt(phys);

    /* every frame starts reserved: holes, the kernel, page tables and mem_map itself
     * must never be handed out, only usable minus reserved is freed below */
    for (uint32_t pfn = 0; pfn < max_pfn; pfn++) {
        struct page *page = pfn_to_page(pfn);
        page->flags = PG_reserved;
        ...
    }
    ...
    /* memblock is not used after this, from here on frames come from alloc_pages */
    phys_mem_for_each_free(free_range);
}
```

`phys_mem_alloc` returns `0x100000` this time. `init_pd` has just freed the loader's page tables, so `mem_map` moves into the first free spot: the old page tables. It has to be in the direct map (all of RAM is), because it is reached through a pointer.

## The buddy system

The buddy system keeps free memory in blocks of 2<sup>order</sup> pages, order 0 to `MAX_ORDER` = 10 (4 KiB to 4 MiB). Every block is aligned to its own size, and there is one free list per order:

```c
/* Number of free blocks, not pages */
struct free_block {
    struct list_head free_list;
    uint32_t nr_free;
};

#define MAX_ORDER 10

/* Order = pow(2, order) number of pages */
struct zone {
    struct free_block free_blocks[MAX_ORDER + 1];
    uint32_t free_pages;
};
```

Only the *first* page of a free block is on a list. It carries `PG_buddy`, and its `order` field records the block size. The list link is `lru`, so the free lists take no memory of their own.

### Who is my buddy?

The commit's note lists three rules for two blocks to be buddies:

1. They have the same order.
2. They are next to each other in physical memory.
3. The lower one is aligned to 2<sup>order+1</sup> pages, so together they form a valid block one order up.

Rules 2 and 3 mean the buddy of the block at `pfn` differs from `pfn` in exactly one bit, bit `order`. Flipping it gives the buddy:

```c
/* Flipping the bit is a clever trick! */
static struct page *neigh_buddy(struct page *page, uint32_t order) {
    uint32_t buddy_pfn = page_to_pfn(page) ^ (1u << order);
    return buddy_pfn < max_pfn ? pfn_to_page(buddy_pfn) : NULL;
}

static void free_one_page(struct page *page, uint32_t order) {
    while (order < MAX_ORDER) {
        struct page *buddy = neigh_buddy(page, order);

        if (!buddy || !(buddy->flags & PG_buddy) || buddy->order != order)
            break;
        remove_from_zone(buddy, order);
        /* the merged block starts at the lower half */
        if (buddy < page)
            page = buddy;
        order++;
    }
    add_to_zone(page, order);
}
```

The test `PG_buddy && order == order` covers rule 1: the buddy must be the head of a free block of the same size, not, say, the middle of a larger free block, or a page in use. Linux's `__free_one_page` does the same with `page_is_buddy()`.

Allocation goes the other way. It takes the smallest non-empty order that is big enough, and while the block is too big, it splits it and puts the upper half back on the list one order down:

```c
struct page *alloc_pages(uint32_t order) {
    for (uint32_t o = order; o <= MAX_ORDER; o++) {
        struct free_block *free_block = &zone.free_blocks[o];
        if (list_empty(&free_block->free_list))
            continue;

        struct page *page = list_first_entry(&free_block->free_list, struct page, lru);
        remove_from_zone(page, o);
        while (o > order) {
            o--;
            add_to_zone(page + (1u << o), o);
        }
        page->refcount = 1;
        zone.free_pages -= (1u << order);
        return page;
    }
    return NULL;
}
```

![Splitting the order-2 block at pfn 0x584 to allocate order 1, then freeing it: 0x584 ^ 2 = 0x586 is a free order-1 buddy, so they merge; 0x584 ^ 4 = 0x580 is a page table, so merging stops](/assets/images/posts/toy-os-6-buddy/split-merge.svg)

*A split and a merge, with the real pfns `kmalloc(5000)` uses in post 7.*

`free_pages` checks what it can before giving a block back: the order is in range, the pfn is aligned to it, the page is not reserved, not already free, and not owned by the slab layer or `kmalloc`, and the refcount is exactly 1.

### Filling the free lists

At boot, each free range from memblock goes through `free_range`, which carves it into the largest aligned blocks that fit:

```c
/* Only those page frames entirely contained are reclaimed */
static void free_range(uint32_t start, uint32_t end) {
    uint32_t pfn = PFN_UP(start);
    uint32_t end_pfn = PFN_DOWN(end);

    while (pfn < end_pfn) {
        /* __builtin_ctz = count trailing zeros */
        uint32_t order = pfn ? MIN((uint32_t)__builtin_ctz(pfn), MAX_ORDER) : MAX_ORDER;
        while (pfn + (1u << order) > end_pfn)
            order--;
        for (uint32_t i = 0; i < (1u << order); i++)
            pfn_to_page(pfn + i)->flags &= ~PG_reserved;
        zone.free_pages += (1u << order);
        free_one_page(pfn_to_page(pfn), order);
        pfn += 1u << order;
    }
}
```

The number of trailing zero bits of `pfn` is the largest order it is aligned to. The inner loop shrinks the block until it fits in what is left. Starting from pfn `0x582` (`...0010`), that gives order 1, then `0x584` order 2, `0x588` order 3, and so on, up to `0x800`, where order 10 takes over. The comment and my third commit note point out the limit of `PFN_UP`/`PFN_DOWN`: a range that starts or ends in the middle of a page loses that partial page. With E820 entries and reservations that are all page aligned, nothing is lost here.

I read the resulting free lists straight out of `zone` in the Bochs debugger. The struct addresses on the lists convert back to pfns:

![The free lists after boot: one block each for orders 0 to 7 except 8, one order-9 block at 0x600, and 126 order-10 blocks from 0x1FC00 down to 0x800](/assets/images/posts/toy-os-6-buddy/free-lists.svg)

*`zone.free_blocks` after `init_page_alloc()`. `list_add` inserts at the head, so the last block freed, the highest, comes first.*

## Physical memory after init_page_alloc()

![Physical memory: low 1 MiB reserved, mem_map from 1 MiB to 0x480000, 129 free pages up to 0x501000, the kernel PD and 128 PTs up to 0x582000, then free RAM to 512 MiB](/assets/images/posts/toy-os-6-buddy/phys-after-buddy.svg)

*The final physical layout. Everything white is now in the buddy allocator. memblock's reserved list ends as `[0, 0x480000)` and `[0x501000, 0x582000)`.*

| Physical | Size | What |
| --- | --- | --- |
| `0x00000000`–`0x000FFFFF` | 1 MiB | reserved: BIOS areas, kernel image, stack (post 4) |
| `0x00100000`–`0x0047FFFF` | 3.5 MiB | `mem_map`, where the loader page tables were |
| `0x00480000`–`0x00500FFF` | 129 pages | free: an order-7 block and one order-0 page |
| `0x00501000`–`0x00501FFF` | 4 KiB | kernel page directory (CR3) |
| `0x00502000`–`0x00581FFF` | 512 KiB | 128 kernel page tables |
| `0x00582000`–`0x1FFFFFFF` | ~506 MiB | free |

131072 frames − 256 (low MiB) − 896 (`mem_map`) − 129 (page tables) = **129791** free pages, which is exactly what the kernel prints:

![Bochs: page allocator: 129791 free pages, mem_map at phys 0x100000](/assets/images/posts/toy-os-6-buddy/page-alloc.png)

*The second line on the screen.*

The page allocator only hands out whole pages, though, and most kernel objects are a few dozen bytes. [Post 7](/blogs/os/toy-os-7-slab.html) builds slab caches and `kmalloc` on top of it.

## References

1. D. P. Bovet and M. Cesati, *Understanding the Linux Kernel*, 3rd ed., ch. 8 "Memory Management": page descriptors, memory zones, high memory, and "The Buddy System Algorithm". Also ch. 2, "Paging in Linux": the kernel's final page tables and the 896 MiB limit.
2. 郑钢, 《操作系统真相还原》, ch. 5 (获取物理内存容量: E820, E801 and 88h) and ch. 8 (内存管理系统). The book manages memory with bitmaps; toy-os follows Linux here.
3. navi, [从零开始写 OS 内核 - 虚拟内存完善](https://segmentfault.com/a/1190000040203948), segmentfault.
4. Linux source: `mm/memblock.c`, and `__free_one_page()` and `__find_buddy_pfn()` in `mm/page_alloc.c`.
5. OSDev Wiki, [Detecting Memory (x86)](https://wiki.osdev.org/Detecting_Memory_(x86)).
