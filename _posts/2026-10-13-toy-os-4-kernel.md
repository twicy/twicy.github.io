---
layout: page
permalink: /blogs/os/toy-os-4-kernel.html
title: "Toy OS 4: Loading the Kernel"
topic: Operating System and Kernel Bypassing
math: false
description: >-
  The fourth toy-os worklog gets from the loader into C. The kernel is a freestanding
  32-bit ELF linked at 0xC0001500. The loader reads it from disk into a buffer at 0x70000,
  copies each PT_LOAD segment to its virtual address, zeroes .bss and jumps to e_entry with
  the stack at 0xC009F000. A map shows every piece of the first MiB at the moment main()
  starts. The post ends with the first output from C: a VGA text driver and a printk built
  on the compiler's varargs builtins.
---

# Toy OS 4: Loading the Kernel

*Toy OS worklog: [1. Real mode](/blogs/os/toy-os-1-real-mode.html) · [2. Segmentation](/blogs/os/toy-os-2-segmentation.html) · [3. Paging](/blogs/os/toy-os-3-paging.html) · **4. Kernel** · [5. Interrupts](/blogs/os/toy-os-5-interrupts.html) · [6. Buddy](/blogs/os/toy-os-6-buddy.html) · [7. Slab](/blogs/os/toy-os-7-slab.html) · [Source](https://github.com/twicy/toy-os/tree/fc88a1de36e8cd2cf495c46f3e6b36c01fc94f8b)*

Paging is on and the loader runs in protected mode with a stack at `0xC009F000` (post 3). The last thing it does is start the kernel. The kernel is written in C and built as an ordinary ELF executable. An ELF file is not a memory image, though, so the loader has to read the file's program headers and lay out the memory itself.

## Table of Contents

- [Building the kernel](#building-the-kernel)
- [Reading kernel.bin from the disk](#reading-kernelbin-from-the-disk)
- [Copying the segments](#copying-the-segments)
- [Physical memory when main() starts](#physical-memory-when-main-starts)
- [First words from C: VGA and printk](#first-words-from-c-vga-and-printk)
- [References](#references)

## Building the kernel

```make
CFLAGS := -m32 -ffreestanding -nostdinc -Iinclude \
          -fno-pie -fno-stack-protector -fno-asynchronous-unwind-tables \
          -Wall -Wextra -MMD -MP

kernel.bin: $(KERNEL_OBJS)
	ld -m elf_i386 $^ -Ttext 0xc0001500 -e main -o $@
```

- `-m32`, `-m elf_i386`: 32-bit code and a 32-bit ELF.
- `-ffreestanding -nostdinc`: there is no C library and no host headers. `include/` has its own `stdint.h`, `stdarg.h` and `string.h`.
- `-fno-pie`: the kernel is linked at a fixed address and nothing will relocate it.
- `-fno-stack-protector`: otherwise gcc emits canary checks. On i386 the canary is read from `%gs:0x14`, a thread-local slot that does not exist here.
- `-fno-asynchronous-unwind-tables`: no `.eh_frame`, since nothing will ever unwind the stack.
- `-Ttext 0xc0001500 -e main`: `.text` starts at `0xC0001500`, and the ELF entry point is `main`.

Why `0xC0001500`? `0xC0000000` + PA `0x1500` puts the kernel in the higher half, in low memory just above the loader (`0x800`–`0xFFF`). As my commit note says, `ld` decides the target virtual addresses, and the ELF program headers record them:

```text
$ readelf -l kernel.bin
Entry point 0xc00017b4
There are 5 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0xc0001000 0xc0000000 0x000d4 0x000d4 R   0x1000
  LOAD           0x000500 0xc0001500 0xc0001500 0x03cd3 0x03cd3 R E 0x1000
  LOAD           0x005000 0xc0006000 0xc0006000 0x00d6c 0x00d6c R   0x1000
  LOAD           0x005d6c 0xc0007d6c 0xc0007d6c 0x00004 0x010a8 RW  0x1000
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x10

 Section to Segment mapping:
  Segment Sections...
   00
   01     .text
   02     .rodata
   03     .data .bss
```

- With 4 KiB alignment, a segment's file offset and virtual address must be equal modulo 4 KiB. `.text` at `0xC0001500` therefore sits at file offset `0x500`. The first page of the file, `0xC0001000`, holds only the ELF header and program headers: segment 00, 212 bytes.
- The entry point `0xC00017B4` is not the start of `.text`. `gdt.o` comes first in the link order, and `-e main` just records `main`'s address.
- The last segment has `FileSiz` 4 but `MemSiz` `0x10A8`: that is `.bss`. It takes up memory but has no bytes in the file.

## Reading kernel.bin from the disk

Before paging (post 3), the loader reads 200 sectors from LBA 9 to a buffer at `0x70000`. It uses the same ATA PIO routine as the MBR (post 1), rewritten with 32-bit registers:

```nasm
    ; load kernel.bin into a buffer, it is an ELF file so its code is not at offset 0
    mov eax, KERNEL_START_SECTOR    ; 9
    mov ebx, KERNEL_BIN_BASE_ADDR   ; 0x70000
    mov ecx, KERNEL_SECTOR_COUNT    ; 200
    call read_disk
```

200 sectors is 100 KiB, so the buffer is `[0x70000, 0x89000)`. Today `kernel.bin` is 34,272 bytes (67 sectors). `KERNEL_SECTOR_COUNT` must match `count=200` in the Makefile, and the kernel file must stay under 100 KiB, or its tail is silently cut off. The first bytes of the buffer show the ELF magic:

```text
<bochs> xp /16bx 0x70000
0x0000000000070000 <bogus+       0>:	0x7f	0x45	0x4c	0x46	0x01	0x01	0x01	0x00
0x0000000000070008 <bogus+       8>:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
```

## Copying the segments

`kernel_init` walks the program header table and handles each `PT_LOAD` entry. The offsets are those of `Elf32_Ehdr` and `Elf32_Phdr`:

| Field | Offset | | Field | Offset |
| --- | --- | --- | --- | --- |
| `e_entry` | 24 | | `p_type` | 0 |
| `e_phoff` | 28 | | `p_offset` | 4 |
| `e_phentsize` | 42 | | `p_vaddr` | 8 |
| `e_phnum` | 44 | | `p_filesz` / `p_memsz` | 16 / 20 |

```nasm
kernel_init:
    ...
    ; e_phentsize, namely program header size
    mov dx, [KERNEL_BIN_BASE_ADDR + 42]
    ; e_phoff, offset of the first program header
    mov ebx, [KERNEL_BIN_BASE_ADDR + 28]
    ; e_phnum, number of program header
    mov cx, [KERNEL_BIN_BASE_ADDR + 44]

    add ebx, KERNEL_BIN_BASE_ADDR

.each_segment:
    ; p_type is a dword, only PT_LOAD segments get copied into memory
    cmp dword [ebx], PT_LOAD
    jne .next_segment

    ; memcpy from bin to the designated place
    push dword [ebx + 16]           ; p_filesz
    mov eax, [ebx + 4]              ; p_offset
    add eax, KERNEL_BIN_BASE_ADDR
    push eax
    push dword [ebx + 8]            ; p_vaddr
    call mem_cpy
    add esp, 12

    ; .bss takes memory but no file bytes (p_memsz > p_filesz), zero
    ; the tail so C globals start at 0
    mov eax, [ebx + 20]
    sub eax, [ebx + 16]
    push eax                        ; p_memsz - p_filesz
    mov eax, [ebx + 8]
    add eax, [ebx + 16]
    push eax                        ; p_vaddr + p_filesz
    call mem_zero
    add esp, 8

.next_segment:
    add ebx, edx
    loop .each_segment
    ret
```

![kernel.bin in the buffer on the left, the loaded kernel in memory on the right: headers, .text, .rodata and .data copied to their virtual addresses, .bss zeroed](/assets/images/posts/toy-os-4-kernel/elf-load.svg)

*The file layout (offsets from `0x70000`) and the memory layout it turns into. The bands are the four `mem_cpy` calls.*

The `p_vaddr` addresses are above `0xC0000000`. That works because paging is on and PDE 768 maps VA `0xC0000000`–`0xC00FFFFF` to the first MiB. As my commit note says, this loader pays for an extra copy: disk → buffer → final place. A smarter loader would read the headers first and then read each segment straight from the disk to its address.

The `.bss` zeroing was not there at first. It came with printk ([`c91a8c0`](https://github.com/twicy/toy-os/commit/c91a8c0)), whose `cursor_x` and `cursor_y` were the first C globals that had to start at 0.

Then the jump:

```nasm
    call kernel_init

    mov esp, KERNEL_STACK_TOP       ; 0xC009F000
    ; e_entry
    jmp [KERNEL_BIN_BASE_ADDR + 24]
```

The kernel stack is the loader's stack, reset to its top: the loader's frames are simply forgotten. At the first instruction of `main`:

```text
(0) Breakpoint 7, 0x00000000c00017b4 in ?? ()
(0) [0x0000000017b4] 0008:00000000c00017b4 (unk. ctxt): lea ecx, ss:[esp+4]       ; 8d4c2404
rsp: 00000000_c009f000
rip: 00000000_c00017b4
```

## Physical memory when main() starts

This is the whole first MiB, plus what lies above it, at the first instruction of `main`:

![Physical memory when main() starts: IVT, BDA, E820 map at 0x500, the dead loader at 0x800, the kernel image at 0x1000-0x8E13, free space, the dead kernel.bin buffer at 0x70000, the kernel stack below 0x9F000, EBDA, video and BIOS, the loader page tables at 1 MiB, then free RAM](/assets/images/posts/toy-os-4-kernel/low-1mib-kernel.svg)

*Physical memory at `main()`. Amber regions are done and could be reused. The whole first MiB is also visible at VA `0xC0000000` + PA.*

| Physical | Virtual | What |
| --- | --- | --- |
| `0x00500`–`0x00567` | `0xC0000500` | E820 map: count + 5 entries (room for 32, up to `0x784`) |
| `0x00800`–`0x00B70` | | `loader.bin`; its GDT at `0x803` is in use until `init_gdt()` (post 2) |
| `0x01000`–`0x010D3` | `0xC0001000` | ELF + program headers |
| `0x01500`–`0x051D2` | `0xC0001500` | `.text` |
| `0x06000`–`0x06D6B` | `0xC0006000` | `.rodata` |
| `0x07D6C`–`0x08E13` | `0xC0007D6C` | `.data` (4 bytes) + `.bss` (GDT, IDT, allocator state, …) |
| `0x70000`–`0x88FFF` | | `kernel.bin` as read from the disk |
| ↓ `0x9F000` | `0xC009F000` | kernel stack, growing down |
| `0xB8000`–`0xBFFFF` | `0xC00B8000` | text buffer |
| `0x100000`–`0x500FFF` | (recursive window, post 3) | loader page directory and page tables |

A few things follow from this layout:

- The kernel image overwrites the MBR (`0x7C00`–`0x7DFF`), which is fine, since the MBR is long dead. The kernel's GDT (post 2) ends up exactly in the MBR's last 128 bytes.
- In memory, the kernel can grow up to the buffer at `0x70000`, about 444 KiB. The file is the tighter limit: 100 KiB.
- Nothing here is reclaimed yet. Post 6 simply reserves the whole first MiB. With a kernel of about 32 KiB, that is simpler than tracking every gap.
- `kernel.h` turns the higher-half rule into two macros, valid for anything in the kernel half:

```c
/* the kernel half starts here, physical address p is at KERNEL_OFFSET + p */
#define KERNEL_OFFSET 0xC0000000
#define phys_to_virt(x) ((void *)((uintptr_t)(x) + KERNEL_OFFSET))
#define virt_to_phys(x) ((uintptr_t)(x) - KERNEL_OFFSET)
```

## First words from C: VGA and printk

The first `main` ([`bf3a03a`](https://github.com/twicy/toy-os/commit/bf3a03a)) was just `while(1);`. Once that ran without faulting, the next commit ([`c91a8c0`](https://github.com/twicy/toy-os/commit/c91a8c0)) gave the kernel a voice.

### The text buffer

In 80 × 25 color text mode, each screen cell is two bytes in the buffer: the character, then an attribute.

![A VGA text cell: bit 15 blink, bits 14-12 background, bit 11 bright, bits 10-8 foreground, bits 7-0 the character; 0x0F74 is a white t on black](/assets/images/posts/toy-os-4-kernel/vga-cell.svg)

*One cell. The kernel prints white on black, attribute `0x0F`.*

`src/vga.c` writes cells at `0xC00B8000`, keeps its own cursor, scrolls by copying rows 1–24 up one row, and moves the hardware cursor through the CRTC registers:

```c
/* 0xB8000 seen through the kernel alias the loader maps at 0xC0000000 */
#define VIDEO_SEG_BASE 0xc00b8000

void put_char(char c, uint8_t color) {
    uint16_t *video_memory = (uint16_t *)VIDEO_SEG_BASE;

    if (c == '\n') {
        cursor_x = 0;
        cursor_y++;
    } else if (c == '\r') {
        cursor_x = 0;
    } else {
        // char is signed, without the cast bytes >= 0x80 sign-extend over the color
        video_memory[cursor_y * VGA_WIDTH + cursor_x] = (color << 8) | (uint8_t)c;
        cursor_x++;
    }
    ...
}
```

The comment marks a real trap. `char` is signed on x86, so a byte like `0xB0` becomes `0xFFB0` when promoted to `int`. OR'ed with `color << 8`, that wipes out the attribute byte.

### printk and varargs

`printk` is a thin wrapper: `vformat()` in `lib/stdio.c` parses the format and calls a `putc` callback for each character. It supports `%d %i %u %o %x %X %c %s %p %%`, the flags `- + space # 0`, width and precision (`*` too), and the length modifiers `hh h l z`.

```c
static void vga_putc(char c, void *arg) {
    (void)arg;
    put_char(c, VGA_COLOR_DEFAULT);
}

/* Characters go straight to the screen, so no buffer limits the length */
int printk(const char *format, ...) {
    va_list ap;
    int len;

    va_start(ap, format);
    len = vformat(vga_putc, NULL, format, ap);
    va_end(ap);
    return len;
}
```

With `-nostdinc` there is no `<stdarg.h>`, but varargs are a compiler feature anyway. `include/stdarg.h` just names the builtins:

```c
typedef __builtin_va_list va_list;
#define va_start(ap, last) __builtin_va_start(ap, last)
#define va_arg(ap, type) __builtin_va_arg(ap, type)
#define va_end(ap) __builtin_va_end(ap)
```

On i386, `__builtin_va_list` is just a `char *` into the caller's stack frame. The cdecl caller pushes the arguments right to left, so in memory they sit left to right above the return address:

```text
c00017d2:	push   $0xc00017b4      ; 2nd argument: main
c00017d7:	push   $0xc0006241      ; 1st argument: the format string
c00017dc:	call   c0001adf <printk>
```

`va_start(ap, format)` points `ap` just past `format`. Each `va_arg(ap, int)` reads 4 bytes and moves on. Arguments narrower than `int` (`char`, `short`) are promoted to `int` when passed, which is why `vformat` reads `%hhd` as `(signed char)va_arg(ap, int)`.

![Bochs: "toy-os kernel entered at 0xc00017b4"](/assets/images/posts/toy-os-4-kernel/kernel-entry.png)

*`main` clears the screen and prints its own address: `%p` of `main` = `e_entry`.*

The kernel can talk now, but it cannot listen: any exception would still take the machine down. [Post 5](/blogs/os/toy-os-5-interrupts.html) adds an IDT, the 8259A PIC and a timer.

## References

1. 郑钢, 《操作系统真相还原》, ch. 5 (加载内核: ELF, the kernel at `0xc0001500`, the `0x70000` buffer) and ch. 6 (完善内核: printing to the screen).
2. navi, [从零开始写 OS 内核 - 加载并进入 kernel](https://segmentfault.com/a/1190000040167201) and [显示与打印](https://segmentfault.com/a/1190000040179784), segmentfault.
3. `man 5 elf`: `Elf32_Ehdr` and `Elf32_Phdr`.
4. System V Application Binary Interface, Intel386 Architecture Processor Supplement: the cdecl calling convention and variable argument lists.
5. OSDev Wiki, [Text UI](https://wiki.osdev.org/Text_UI) and [Text Mode Cursor](https://wiki.osdev.org/Text_Mode_Cursor).
