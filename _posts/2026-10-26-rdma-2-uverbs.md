---
layout: page
permalink: /blogs/os/rdma-2-uverbs.html
title: "RDMA Deep Dive 2: uverbs, ucontext, uobject and umem"
date: 2026-10-26 12:00:00 -0700
topic: Operating System and Kernel Bypassing
math: false
description: >-
  The second part of the RDMA deep dive lays out the kernel side of the control path. It
  covers the modules (ib_uverbs, ib_uverbs_support, ib_core, mlx5_ib, mlx5_core) and defines
  the words they use: uverbs, ucontext, uobject, handle, uapi tree, udata, umem and the mlx5
  terms. User space reaches the kernel through write(), ioctl() and mmap() on
  /dev/infiniband/uverbsN. Both command ABIs dispatch through one radix tree of methods. The
  integer handle user space passes is an index into a per-file XArray, which yields an
  ib_uobject that points to the driver's object. The post ends with the life of a uobject and
  what ibv_open_device sets up: the ufile, the ucontext and the mmap()ed doorbell pages.
---

# RDMA Deep Dive 2: uverbs, ucontext, uobject and umem

*RDMA deep dive: [1. RC send/receive](/blogs/os/rdma-1-rc-send-recv.html) · **2. uverbs, ucontext, uobject, umem** · [3. Control plane: reg_mr and create_qp](/blogs/os/rdma-3-control-plane.html)*

[Part 1](/blogs/os/rdma-1-rc-send-recv.html) ended with a split: setup verbs are system calls on `cmd_fd`, posting and polling are not. This part is about what sits behind `cmd_fd`: the modules, the vocabulary, and how one call is dispatched. In particular, it shows how the small integer that user space passes as "the PD" becomes a kernel object.

Code references are to Linux at [`d135ab453a74`](https://github.com/torvalds/linux/tree/d135ab453a7400a34930bdda1ef4bdc8acfac80e) and rdma-core at [`07ce3f5`](https://github.com/linux-rdma/rdma-core/tree/07ce3f50f6e2924e33f12a1fdfb0a610d2d07f82). Paths without a prefix are under `drivers/infiniband/`.

## Table of Contents

- [The layers](#the-layers)
- [Vocabulary](#vocabulary)
  - [Verbs objects in the kernel](#verbs-objects-in-the-kernel)
  - [The uverbs side](#the-uverbs-side)
  - [Memory](#memory)
  - [mlx5 words](#mlx5-words)
- [Three system calls: write(), ioctl(), mmap()](#three-system-calls-write-ioctl-mmap)
- [Dispatching one ioctl](#dispatching-one-ioctl)
- [From an int to a kernel object](#from-an-int-to-a-kernel-object)
- [The life of a uobject](#the-life-of-a-uobject)
- [What ibv_open_device sets up](#what-ibv_open_device-sets-up)
- [umem in one picture](#umem-in-one-picture)
- [References](#references)

## The layers

![The software stack from top to bottom: your program, libibverbs, the libmlx5 provider, then /dev/infiniband/uverbs0, then ib_uverbs.ko, ib_uverbs_support.ko, ib_core.ko, mlx5_ib.ko and mlx5_core.ko, then PCIe, then the ConnectX firmware and HCA. The control path goes through every layer; the data path goes from libmlx5 straight to the HCA](/assets/images/posts/rdma-2-uverbs/stack.svg)

*The colours are used again in the call ladders below and in part 3.*

| Layer and source | Owns |
| --- | --- |
| `libibverbs`<br>rdma-core `libibverbs/` | The `ibv_*` API. It turns each control verb into a `write()` or `ioctl()` command and finds the provider for a device. |
| `libmlx5`<br>rdma-core `providers/mlx5/` | The user-space half of the mlx5 driver: WQE and CQE formats, doorbells, and the driver-private part of each command. The data path lives here. |
| `ib_uverbs.ko`<br>`core/uverbs_*.c` | The character device `/dev/infiniband/uverbsN`, parsing of both command ABIs, and the handler of each verb. |
| `ib_uverbs_support.ko`<br>`core/rdma_core.c` | The uobject machinery (handles, lookup, locking, cleanup). It was split out of `ib_uverbs` in 2026 so that drivers can call it without depending on `ib_uverbs` ([`2274d8cb4912`](https://github.com/torvalds/linux/commit/2274d8cb4912)). |
| `ib_core.ko`<br>`core/device.c`, `verbs.c`, `umem.c`, … | `ib_device` registration, the in-kernel verbs (`ib_create_qp_user()`, `ib_modify_qp_with_udata()`), the GID cache, pinning user memory (`umem`), MR pools, resource tracking. |
| `mlx5_ib.ko`<br>`hw/mlx5/` | The `ib_device_ops` for ConnectX: maps PD, CQ, QP and MR onto mlx5 objects and builds their firmware commands. |
| `mlx5_core.ko`<br>`drivers/net/ethernet/`<br>`mellanox/mlx5/core/` | The PCI driver: firmware command queue, event queues (EQs), UAR pages, health. It also contains the Ethernet driver (`mlx5e`). |

**A note on names.** The NVIDIA (formerly Mellanox) driver is often called "the mlnx driver". Upstream it is two modules: `mlx5_core` owns the PCI function and is shared by Ethernet, RDMA and others, and `mlx5_ib` is the RDMA part. `mlx5_ib` binds to an *auxiliary device* named `mlx5_core.rdma.<n>`, which `mlx5_core` creates (`mlx5r_probe()` in `hw/mlx5/main.c`). ConnectX-3 and older use the same split with `mlx4_core` and `mlx4_ib`. MLNX_OFED (now DOCA-OFED) is NVIDIA's out-of-tree package of these drivers.

## Vocabulary

### Verbs objects in the kernel

| Term | Definition |
| --- | --- |
| **verbs** | The abstract operations of the InfiniBand specification (ch. 11): allocate a PD, register memory, create a QP, post a WR, poll a CQ. `libibverbs` implements them for user space. `ib_core` implements them for kernel users such as NVMe-oF, iSER, SMB Direct and NFS/RDMA. |
| **`struct ib_device`** | One RDMA device as `ib_core` sees it (`mlx5_0`). The driver fills in a `struct ib_device_ops` table (`mlx5_ib_dev_ops`: `.create_qp = mlx5_ib_create_qp`, `.reg_user_mr = mlx5_ib_reg_user_mr`, …) and calls `ib_register_device()`. |
| **core and driver objects** | `ib_pd`, `ib_cq`, `ib_qp`, `ib_mr` are the generic parts. Each driver embeds them in its own struct (`mlx5_ib_pd { struct ib_pd ibpd; u32 pdn; … }`) and gets back to it with `container_of()`. For most types, `ib_core` allocates the whole driver struct, sized by `INIT_RDMA_OBJ_SIZE(ib_qp, mlx5_ib_qp, ibqp)`, via `rdma_zalloc_drv_obj()`. The driver only initializes it. MRs are the exception: `reg_user_mr` allocates and returns its own. |
| **`usecnt`** | A counter in `ib_pd`, `ib_cq`, … of the objects that depend on it. Destroying an object whose `usecnt` is non-zero fails with `-EBUSY`. |

### The uverbs side

| Term | Definition |
| --- | --- |
| **uverbs** | *User verbs*: the kernel interface that lets user space create and control verbs objects. Code in `core/uverbs_*.c`, built as `ib_uverbs.ko`. |
| **`struct ib_uverbs_device`** | One per `ib_device`. It owns the char device `/dev/infiniband/uverbsN` and the device's uapi tree. |
| **ufile** (`struct ib_uverbs_file`) | Created by each `open()` of the char device; `filp->private_data` points to it. It holds the handle table (`idr`), the list of live uobjects, the ucontext, and the `mmap()`ed regions. Everything a process creates belongs to a ufile. |
| **ucontext** (`struct ib_ucontext`) | The per-file device context that the driver sets up: for mlx5, the UAR pages, the transport domain and the doorbell-record pages. It is created by the `GET_CONTEXT` command that `ibv_open_device()` sends. There is one per ufile, and no other object can be created before it exists. |
| **uobject** (`struct ib_uobject`) | The kernel's record of one object created from user space: its handle (`id`), its type, a lock (`usecnt`), a refcount, its owner (`ufile`, `context`), and `object`, a pointer to the real `ib_pd`/`ib_qp`/…. Not to be confused with a `kobject`, the driver model's sysfs object. Most types are *IDR-class*, found by handle. Completion channels and async-event files are *FD-class*: their handle is a file descriptor. |
| **handle** | The `u32` that user space uses to name an object in later commands (`ibv_pd->handle`). The kernel hands it out at creation: it is the uobject's index in the ufile's XArray, and means nothing in another file. |
| **uapi tree** (`struct uverbs_api`) | A per-device radix tree of *objects → methods → attributes* that every command is looked up in. It is built from the core definitions plus the driver's (`ib_device->driver_def`), so drivers can add their own objects, such as mlx5's DEVX and UAR objects. |
| **method, attribute** | An ioctl command is a *method* on an *object* (`QP` / `QP_CREATE`) with a list of typed *attributes*. Each attribute is declared with a spec: `UVERBS_ATTR_IDR(…, UVERBS_OBJECT_PD, UVERBS_ACCESS_READ, …)`. |
| **attribute bundle** (`struct uverbs_attr_bundle`) | The parsed arguments of one call, passed to the handler: the attributes, the ufile and context, `ucore` (the core command data) and `driver_udata`. |
| **udata** (`struct ib_udata`) | A pair of user buffers (in, out) for data that only the driver understands. libmlx5 sends `struct mlx5_ib_create_qp { buf_addr, db_addr, sq_wqe_count, … }` and gets `bfreg_index` back. The core passes it through untouched. In the ioctl ABI these are the `UHW_IN`/`UHW_OUT` attributes ("user hardware"). |
| **mmap entry** (`struct rdma_user_mmap_entry`) | A driver-registered page range that user space can map. The kernel hands out an opaque offset, and `mmap(…, cmd_fd, offset)` maps it. This is how doorbell (UAR) pages reach the process. |
| **disassociate** | When the device goes away (hot unplug, driver reset), uverbs destroys the hardware objects behind open files. Commands then fail with `-EIO`, but the process keeps its file descriptors and can still destroy its objects. |

### Memory

| Term | Definition |
| --- | --- |
| **MR, lkey, rkey** | A memory region: a range of virtual memory the NIC may access, with access rights. `lkey` names it in local SGEs, `rkey` in remote requests. On mlx5, both are the same 32-bit *mkey*. |
| **umem** (`struct ib_umem`) | User memory made usable by the device. The range is pinned (`pin_user_pages_fast(FOLL_LONGTERM)`) and DMA-mapped, and the result is a scatter-gather table of DMA addresses. Created by the `ib_umem_get_*()` helpers in `core/umem.c`. MRs use it, and so do the QP, CQ and SRQ rings and the doorbell records that live in user memory. |
| **`ib_umem_odp`** | *On-demand paging*: a umem whose pages are not pinned. The NIC faults on missing pages, and MMU notifiers invalidate its mappings when the kernel moves or frees a page. |
| **`ib_umem_dmabuf`** | A umem built from a dma-buf file descriptor (for example GPU memory) instead of a virtual address range. |

### mlx5 words

| Term | Definition |
| --- | --- |
| **command interface** | How `mlx5_core` talks to firmware: a command (opcode plus object context) goes into a host-memory command queue and a doorbell write starts it. `mlx5_cmd_exec()` does the round trip. Every create, modify and destroy is one. |
| **QPC, CQC, mkc** | The firmware's *contexts* for a QP, CQ or mkey. The driver fills in their fields with `MLX5_SET()` in a create or modify command. The layouts are in `include/linux/mlx5/mlx5_ifc.h`. |
| **ICM** | Interconnect context memory: host memory that the firmware owns and stores contexts in. The driver gives it pages and never reads them. |
| **mkey, MTT** | A *memory key*: index plus 8-bit variant, with PD, start address, length, access rights and a page size. Its translation table, the MTT, holds one DMA address per page. |
| **PAS** | Physical address list: the DMA addresses of a buffer's pages, carried in a create command (for the QP or CQ ring, or the MTTs of an mkey). |
| **UMR** | *User-mode memory registration*: a special WQE posted on a kernel-owned QP that updates an mkey (translations, PD, access, enable) without a firmware command. |
| **UAR, bfreg, BlueFlame** | A *user access region* is a 4 KiB page of NIC registers, mapped into the process. It holds doorbell registers, also called bfregs. BlueFlame writes a small WQE straight into the register with write-combining, which saves the NIC a DMA read. |
| **doorbell record** | 8 bytes in user memory with the SQ and RQ producer counters. The NIC reads it by DMA, so it is pinned too. |
| **uid** (`devx_uid`) | A firmware user ID per ucontext. Objects created with it are checked by the firmware to belong to that user. |

## Three system calls: write(), ioctl(), mmap()

Each `ib_device` gets a character device. `ib_uverbs_add_one()` creates it when the device registers, and its `file_operations` are the whole user-space interface (`core/uverbs_main.c`):

```c
static const struct file_operations uverbs_mmap_fops = {
	.write   = ib_uverbs_write,      /* legacy commands  */
	.mmap    = ib_uverbs_mmap,       /* doorbells, clock */
	.open    = ib_uverbs_open,
	.release = ib_uverbs_close,
	.unlocked_ioctl = ib_uverbs_ioctl,  /* RDMA_VERBS_IOCTL */
	/* … */
};
```

- **`write()`** is the original ABI. The buffer starts with `struct ib_uverbs_cmd_hdr { u32 command; u16 in_words; u16 out_words; }` and a fixed struct per command. A `write()` cannot return data, so the command carries a `response` pointer that the kernel writes the answer to. Driver data follows the core struct. There is an extended form with a second header. No new `write()` commands are being added; new functionality goes into the ioctl interface.
- **`ioctl(cmd_fd, RDMA_VERBS_IOCTL, &hdr)`** is the newer ABI. A header names the object and method, and an array of self-describing attributes follows. The kernel checks each attribute against a spec, which makes optional and future fields easy to add:

![Layout of the ioctl buffer for ibv_create_qp: a 24-byte ib_uverbs_ioctl_hdr with length, object_id QP, method_id QP_CREATE, num_attrs, driver_id MLX5; then 16-byte ib_uverbs_attr entries. The handle attributes for the new QP, the PD and the CQs carry small integers; others point at user buffers](/assets/images/posts/rdma-2-uverbs/ioctl-hdr.svg)

- **`mmap(…, cmd_fd, offset)`** maps driver pages into the process, mainly UAR (doorbell) pages. `ib_uverbs_mmap()` hands it to `ucontext->device->ops.mmap`.

Which ABI a verb uses is up to `libibverbs`. With rdma-core `07ce3f5`, the verbs from part 1 use:

| Verb | ABI | Kernel entry (`core/`) |
| --- | --- | --- |
| `ibv_open_device` → `GET_CONTEXT` | `ioctl` (`write` fallback) | `UVERBS_METHOD_GET_CONTEXT` (`uverbs_std_types_device.c`) |
| `ibv_alloc_pd` | `write` | `ib_uverbs_alloc_pd` (`uverbs_cmd.c`) |
| `ibv_reg_mr` | `write` | `ib_uverbs_reg_mr` (`uverbs_cmd.c`) |
| `ibv_reg_mr_ex`, `ibv_reg_dmabuf_mr` | `ioctl` | `UVERBS_METHOD_REG_MR`, `…_REG_DMABUF_MR` (`uverbs_std_types_mr.c`) |
| `ibv_create_cq` | `ioctl` (`write` fallback) | `UVERBS_METHOD_CQ_CREATE` (`uverbs_std_types_cq.c`) |
| `ibv_create_qp` | `ioctl` (`write` fallback) | `UVERBS_METHOD_QP_CREATE` (`uverbs_std_types_qp.c`) |
| `ibv_modify_qp` | `write` | `ib_uverbs_modify_qp` (`uverbs_cmd.c`) |

The two ABIs do not have separate dispatchers. `uverbs_alloc_api()` builds a single radix tree per device, and the `write()` commands are stored in it as special methods (`UVERBS_API_METHOD_IS_WRITE`) keyed by command number. `ib_uverbs_write()` finds its handler with `uapi_get_method(uapi, hdr.command)`. `ib_uverbs_ioctl()` looks up `(object_id, method_id)`.

## Dispatching one ioctl

![Call ladder of ibv_create_qp through the ioctl path: libibverbs builds the header and attributes; ib_uverbs_ioctl checks the header; ib_uverbs_cmd_verbs looks up the method in the radix tree; ib_uverbs_run_method copies and parses each attribute; for handle attributes uverbs_get_uobject_from_file and rdma_lookup_get_uobject find and lock the uobject; then the method handler runs and calls into ib_core; bundle_destroy commits the new object and unlocks the others](/assets/images/posts/rdma-2-uverbs/ioctl-dispatch.svg)

*Each row is one function. The chip names its module, the indent is the call depth, and the orange rows are the handle lookups described next.*

The specs are what make the parsing generic. `QP_CREATE` declares, among others (`core/uverbs_std_types_qp.c`):

```c
/* the new QP */
UVERBS_ATTR_IDR(UVERBS_ATTR_CREATE_QP_HANDLE,
		UVERBS_OBJECT_QP, UVERBS_ACCESS_NEW, UA_MANDATORY),
/* an existing PD */
UVERBS_ATTR_IDR(UVERBS_ATTR_CREATE_QP_PD_HANDLE,
		UVERBS_OBJECT_PD, UVERBS_ACCESS_READ, UA_OPTIONAL),
```

`uverbs_process_attr()` switches on the spec type. A `PTR_IN` attribute of 8 bytes or less sits inline in `data`; a longer one is a user pointer, copied in now or when the handler reads it. An `IDR` attribute goes to `uverbs_get_uobject_from_file()` with the spec's access mode (`NEW`, `READ`, `WRITE` or `DESTROY`). For `NEW`, the kernel writes the freshly reserved handle straight back into the user's attribute array with `put_user()`. When the handler returns, `bundle_destroy()` walks every object attribute: it commits or aborts the new uobject and releases the locks on the others.

## From an int to a kernel object

The rule is simple: **user space passes an integer handle, and the kernel looks it up in an XArray to get the object**. More precisely:

- The integer is the `u32` handle the kernel returned when the object was created: `resp.mr_handle` for `REG_MR`, or the value written into the `NEW` attribute.
- The XArray is `ufile->idr`, one per open file. It is still called `idr` because it used to be an IDR.
- What comes out is a `struct ib_uobject`, not a `kobject`. Its `object` field points to the verbs object (`struct ib_pd`), which is embedded in the driver's `struct mlx5_ib_pd`. Only the driver struct holds the number the firmware knows, the `pdn`.

![Handle lookup: user space holds ibv_pd with handle 1 and sends the attribute PD_HANDLE with data 1; the kernel does xa_load on the ufile's idr at index 1 and gets an ib_uobject whose id is 1, type PD, with usecnt, kref, owner and an object pointer to the ib_pd embedded in mlx5_ib_pd, which carries pdn 0x9; the ib_pd points back to the uobject](/assets/images/posts/rdma-2-uverbs/handle.svg)

The lookup itself is a few lines (`core/rdma_core.c`):

```c
static struct ib_uobject *
lookup_get_idr_uobject(const struct uverbs_api_object *obj,
		       struct ib_uverbs_file *ufile, s64 id,
		       enum rdma_lookup_mode mode)
{
	struct ib_uobject *uobj;

	if (id < 0 || id > ULONG_MAX)
		return ERR_PTR(-EINVAL);

	rcu_read_lock();
	uobj = xa_load(&ufile->idr, id);
	if (!uobj || !kref_get_unless_zero(&uobj->ref))
		uobj = ERR_PTR(-ENOENT);
	rcu_read_unlock();
	return uobj;
}
```

The caller, `rdma_lookup_get_uobject()`, then checks three more things before the handler sees the pointer:

1. **Type.** `uobj->uapi_object` must be the type the spec asked for, so a CQ handle passed as a PD fails with `-EINVAL`.
2. **Device still there.** If the device was disassociated, everything except destroy fails with `-EIO`.
3. **Lock.** `uverbs_try_lock_object()` takes the uobject's `usecnt` lock:

| `usecnt` | Meaning | `READ` (shared) | `WRITE` / destroy (exclusive) |
| --- | --- | --- | --- |
| `0` | idle | `+1` | `0 → −1` |
| `n > 0` | `n` commands reading it | `+1` | `-EBUSY` |
| `−1` | being written, destroyed, or still being created | `-EBUSY` | `-EBUSY` |

The lock never sleeps. Two threads racing to destroy the same QP get one success and one `-EBUSY`, not a deadlock. A destroy that comes from a `DESTROY` attribute skips this lock (`UVERBS_LOOKUP_DESTROY`); `uobj_destroy()` takes the object exclusively itself.

The `write()` handlers do the same lookup through macros. `ib_uverbs_reg_mr()` calls `uobj_get_obj_read(pd, UVERBS_OBJECT_PD, cmd.pd_handle, attrs)`, which expands to `rdma_lookup_get_uobject(…, UVERBS_LOOKUP_READ, …)` and returns `uobj->object` cast to `struct ib_pd *`.

FD-class objects take the other branch, `lookup_get_fd_uobject()`. The handle is a file descriptor, so the kernel calls `fget()` on it and then checks that the file has the right `f_op` and belongs to the same ufile.

## The life of a uobject

![uobject life: alloc_begin reserves an XArray slot holding NULL with usecnt -1; the handler creates the hardware object; commit stores the uobject in the slot, sets usecnt to 0; in use, READ locks add one and WRITE locks swap 0 for -1; destroy removes the hardware object, erases the slot and frees the uobject after an RCU grace period. On error the handler path goes to alloc_abort](/assets/images/posts/rdma-2-uverbs/uobj-life.svg)

- **`alloc_begin`** (`rdma_alloc_begin_uobject()`) allocates the uobject with `usecnt = −1` and reserves a slot with `xa_alloc(&ufile->idr, &uobj->id, NULL, …)`. The slot holds `NULL`, so the handle exists but a lookup on it finds nothing: no other thread can use a half-built object.
- **The handler** creates the real object and sets `uobj->object`.
- **`commit`** (`rdma_alloc_commit_uobject()`) puts the uobject on `ufile->uobjects`, sets `usecnt = 0`, and replaces the `NULL` with `xa_store()`. From then on the handle works. On failure, `alloc_abort` destroys whatever the handler built and erases the slot.
- **`destroy`** (`uverbs_destroy_uobject()`) asks the type to destroy the hardware object, which fails with `-EBUSY` if something still uses it, then erases the slot. The memory is freed with `kfree_rcu()`, because a concurrent `xa_load()` under RCU may still be looking at it.

When the process closes `cmd_fd` or exits, `uverbs_destroy_ufile_hw()` empties the list. New uobjects are added at the head, so it destroys the newest first. An object that is still in use (a PD with an MR, say) fails, and the loop runs again until the list is empty. That is why a crashed RDMA process does not leak QPs, MRs or pinned memory.

## What ibv_open_device sets up

![The data structures behind cmd_fd: the process's fd points to a struct file whose private_data is the ib_uverbs_file; the ufile points to the ib_uverbs_device (char dev and uapi tree) and through it to the ib_device mlx5_0; its ucontext is an mlx5_ib_ucontext with the embedded ib_ucontext, UAR and bfreg info, transport domain, devx uid and doorbell page list; its idr XArray holds the uobjects by handle; its umaps track the mmap()ed UAR pages](/assets/images/posts/rdma-2-uverbs/ufile.svg)

And this is how they are built:

![Call ladder of ibv_open_device: open() creates the ib_uverbs_file; libmlx5's mlx5_alloc_context sends GET_CONTEXT by ioctl; the handler allocates and initializes the ucontext, mlx5_ib_alloc_ucontext allocates a transport domain but no UARs, because libmlx5 asked for dynamic UARs; later, at the first create_qp, libmlx5 asks for a UAR with a driver-specific ioctl, mlx5_ib runs ALLOC_UAR and registers an mmap entry, and libmlx5 maps it with mmap on cmd_fd as write-combining memory](/assets/images/posts/rdma-2-uverbs/open-device.svg)

1. **`open()`** creates the ufile and an empty handle XArray (`ib_uverbs_open()`). There is no ucontext yet, so any create command fails with `-EINVAL` (`ib_uverbs_get_ucontext_file()`).
2. **`GET_CONTEXT`** makes the ucontext. `ib_init_ucontext()` makes sure the file has none yet, charges the RDMA cgroup, and calls `ops.alloc_ucontext`. For mlx5 that allocates a transport domain (used by the context's raw Ethernet and RSS QPs) and, if the user asked for DEVX, a firmware `uid`. The response tells libmlx5 the CQE version, the maximum WQE sizes, and so on. Then `smp_store_release(&file->ucontext, ucontext)` publishes it.
3. **UAR pages.** Current libmlx5 sets `MLX5_LIB_CAP_DYN_UAR`, so the kernel allocates no UARs up front. The first `create_qp` asks for one with a *driver* method, `MLX5_IB_METHOD_UAR_OBJ_ALLOC`. This is a uverbs object that mlx5 adds to the uapi tree, and it runs the firmware command `ALLOC_UAR`. The kernel registers the page as an mmap entry and returns an offset. libmlx5 then calls `mmap()` with that offset, and `rdma_user_mmap_io()` maps the UAR page as write-combining memory. That page is where `ibv_post_send()` rings its doorbell.

## umem in one picture

A umem is the answer to "the NIC needs to DMA into memory that `malloc()` gave the process":

```c
struct ib_umem {
	struct ib_device       *ibdev;
	struct mm_struct       *owning_mm;  /* charged in pinned_vm */
	u64                     iova;       /* address the HCA uses */
	size_t                  length;
	unsigned long           address;    /* user virtual address */
	/* … dma_attrs, dma_dir, writable, is_odp, is_dmabuf … */
	struct sg_append_table  sgt_append; /* pages → DMA addresses */
};
```

The same helper pins an MR's buffer, a QP's WQE ring, a CQ's CQE ring and the page that holds a doorbell record. [Part 3](/blogs/os/rdma-3-control-plane.html) goes through `ib_umem_get_va()` line by line, then follows `ibv_reg_mr()` and `ibv_create_qp()` to their firmware commands.

## References

1. Linux, `drivers/infiniband/core/`: `uverbs_main.c` (file operations), `uverbs_ioctl.c` (ioctl parsing), `uverbs_uapi.c` (the method tree), `rdma_core.c` (uobjects), `uverbs_cmd.c` (`write()` handlers), `uverbs_std_types_*.c` (ioctl methods), `umem.c`; `include/rdma/uverbs_types.h`, `include/rdma/ib_verbs.h`, `include/uapi/rdma/rdma_user_ioctl_cmds.h`.
2. Linux, `drivers/infiniband/hw/mlx5/main.c`: `mlx5_ib_dev_ops`, `mlx5_ib_alloc_ucontext()`, `mlx5_ib_mmap()`, the UAR object.
3. rdma-core, `libibverbs/cmd_*.c` (command marshalling, ioctl with write fallback) and `providers/mlx5/` (`mlx5_alloc_context`, `mlx5_alloc_dyn_uar`).
4. Linux kernel documentation, [InfiniBand and Remote DMA (RDMA) interfaces](https://docs.kernel.org/driver-api/infiniband.html) and [Userspace verbs access](https://docs.kernel.org/infiniband/user_verbs.html).
