---
layout: page
permalink: /blogs/os/rdma-3-control-plane.html
title: "RDMA Deep Dive 3: reg_mr and create_qp, Down to the Firmware"
date: 2026-10-26 13:00:00 -0700
topic: Operating System and Kernel Bypassing
math: false
description: >-
  The third part of the RDMA deep dive follows ibv_reg_mr and ibv_create_qp from libibverbs
  to mlx5 firmware commands, and marks which module does each step. Every control verb has
  the same shape: ib_uverbs resolves handles, ib_core runs the common code, mlx5_ib builds
  the hardware object, and mlx5_core sends the firmware command. Registering memory pins the
  pages with pin_user_pages_fast(FOLL_LONGTERM), charged against RLIMIT_MEMLOCK, and
  DMA-maps them. It then picks the largest usable page size, takes an mkey from ib_core's
  new FRMR pools (creating one with CREATE_MKEY only on a miss), and fills it with a UMR work
  request. Creating a QP pins the user-space WQE ring and doorbell record and passes their
  DMA addresses to CREATE_QP. modify_qp resolves RoCE addresses and maps each state change
  to a firmware opcode.
---

# RDMA Deep Dive 3: reg_mr and create_qp, Down to the Firmware

*RDMA deep dive: [1. RC send/receive](/blogs/os/rdma-1-rc-send-recv.html) · [2. uverbs, ucontext, uobject, umem](/blogs/os/rdma-2-uverbs.html) · **3. Control plane: reg_mr and create_qp***

[Part 1](/blogs/os/rdma-1-rc-send-recv.html) wrote the program and [part 2](/blogs/os/rdma-2-uverbs.html) named the pieces behind `cmd_fd`. This part follows the two most interesting setup verbs, `ibv_reg_mr()` and `ibv_create_qp()`, from `libibverbs` to the firmware commands. Each step is labelled with the module it runs in. The call paths are drawn as ladders: one row per function, a coloured chip for its module, the indent for call depth. They leave out locking, error unwinding and rarely used branches, but keep every step that does real work.

Code references are to Linux at [`d135ab453a74`](https://github.com/torvalds/linux/tree/d135ab453a7400a34930bdda1ef4bdc8acfac80e) and rdma-core at [`07ce3f5`](https://github.com/linux-rdma/rdma-core/tree/07ce3f50f6e2924e33f12a1fdfb0a610d2d07f82). Paths without a prefix are under `drivers/infiniband/`.

## Table of Contents

- [The shape of a control verb](#the-shape-of-a-control-verb)
- [How mlx5 runs a firmware command](#how-mlx5-runs-a-firmware-command)
- [ibv_reg_mr](#ibv_reg_mr)
  - [User space (rows 1–4)](#user-space-rows-14)
  - [uverbs: handles first (rows 5–8)](#uverbs-handles-first-rows-58)
  - [Pinning the pages (rows 10–12)](#pinning-the-pages-rows-1012)
  - [Choosing a page size (rows 14–15)](#choosing-a-page-size-rows-1415)
  - [The mkey: pool, firmware, UMR (rows 16–19)](#the-mkey-pool-firmware-umr-rows-1619)
  - [Back to user space, and back again](#back-to-user-space-and-back-again)
- [ibv_create_qp](#ibv_create_qp)
  - [libmlx5 allocates the rings (rows 1–6)](#libmlx5-allocates-the-rings-rows-16)
  - [uverbs and ib_core (rows 7–10)](#uverbs-and-ib_core-rows-710)
  - [mlx5_ib pins the rings and builds the QPC (rows 11–19)](#mlx5_ib-pins-the-rings-and-builds-the-qpc-rows-1119)
  - [What the QP looks like afterwards](#what-the-qp-looks-like-afterwards)
- [ibv_modify_qp, briefly](#ibv_modify_qp-briefly)
- [Who owns what](#who-owns-what)
- [References](#references)

## The shape of a control verb

Every control verb takes the same four steps through the kernel. Only the middle two change from verb to verb:

1. **`ib_uverbs`** parses the command, turns every input handle into a locked object, and reserves a handle for the new one (part 2).
2. **`ib_core`** runs the generic part. It allocates the object (for most types), checks rules that hold for every device, and offers shared helpers: pinning user memory, MR pools, GID and MAC resolution.
3. **The driver** (`mlx5_ib`) implements the `ib_device_ops` hook. It translates the verbs object into mlx5 objects and builds a firmware command.
4. **`mlx5_core`** sends the command to the firmware.

Then the steps unwind: `ib_uverbs` publishes the new handle and copies the response out.

## How mlx5 runs a firmware command

All mlx5 firmware commands go through `mlx5_cmd_exec()` in `mlx5_core` (`drivers/net/ethernet/mellanox/mlx5/core/cmd.c`):

![mlx5 command path: the input mailbox with opcode and object context goes into a command queue entry in host memory; mlx5_core writes a bit to the cmd_dbell register in BAR0; the firmware DMA-reads, executes and writes the output and status; a completion EQE raises an interrupt handled by mlx5_cmd_comp_handler, which wakes wait_func; mlx5_cmd_check turns status and syndrome into an errno](/assets/images/posts/rdma-3-control-plane/fw-cmd.svg)

A driver builds the input with the `MLX5_SET()` macros and calls it. `mlx5_core_create_mkey()` (`mlx5/core/mr.c`), used below, is the whole pattern in a dozen lines:

```c
int mlx5_core_create_mkey(struct mlx5_core_dev *dev, u32 *mkey, u32 *in,
			  int inlen)
{
	u32 lout[MLX5_ST_SZ_DW(create_mkey_out)] = {};
	u32 mkey_index;
	int err;

	MLX5_SET(create_mkey_in, in, opcode, MLX5_CMD_OP_CREATE_MKEY);

	err = mlx5_cmd_exec(dev, in, inlen, lout, sizeof(lout));
	if (err)
		return err;

	mkey_index = MLX5_GET(create_mkey_out, lout, mkey_index);
	*mkey = MLX5_GET(create_mkey_in, in, memory_key_mkey_entry.mkey_7_0) |
		mlx5_idx_to_mkey(mkey_index);
	return 0;
}
```

The firmware picks the index (bits 31–8 of the key); the driver chose the low byte, the *variant*. The other helpers, `mlx5_qpc_create_qp()`, `mlx5_core_qp_modify()` and `mlx5_cmd_uar_alloc()`, look the same: set an opcode, `mlx5_cmd_exec()`, read the new object's number out of the output.

## ibv_reg_mr

![Call ladder of ibv_reg_mr: libibverbs and libmlx5 send REG_MR with write(); ib_uverbs_write dispatches to ib_uverbs_reg_mr, which reserves a handle and read-locks the PD; mlx5_ib_reg_user_mr calls ib_core's ib_umem_get_va to pin pages with pin_user_pages_fast and DMA-map them; create_real_mr picks a page size, pops an mkey from the ib_core FRMR pool, which on a miss calls back into mlx5r_create_mkeys and mlx5_core's CREATE_MKEY; mlx5r_umr_update_mr_pas loads the translations with a UMR work request; the handle is committed and lkey and rkey returned](/assets/images/posts/rdma-3-control-plane/reg-mr.svg)

### User space (rows 1–4)

`ibv_reg_mr(pd, addr, length, access)` is a macro for `ibv_reg_mr_iova2()` with `iova = addr`, the address the NIC will use for this buffer. It is normally the same as the CPU's.

`ibv_dontfork_range()` exists for an old problem. After `fork()`, both processes share the pages copy-on-write. If the parent then writes first, it gets a fresh copy, and the NIC keeps DMAing into the old page, which now belongs to the child. Old kernels needed `madvise(MADV_DONTFORK)` on every registered range. Current kernels copy DMA-pinned anonymous pages for the child at `fork()` time (`folio_needs_cow_for_dma()`) and say so over RDMA netlink (`RDMA_NLDEV_SYS_ATTR_COPY_ON_FORK`). On those kernels, libibverbs reports `IBV_FORK_UNNEEDED` and the call does nothing.

The provider hook `mlx5_reg_mr()` has nothing mlx5-specific to add, so it calls the generic `ibv_cmd_reg_mr()`. That sends `IB_USER_VERBS_CMD_REG_MR` with `write()`: `{ start, length, hca_va, pd_handle, access_flags }`.

### uverbs: handles first (rows 5–8)

`ib_uverbs_write()` finds the handler `ib_uverbs_reg_mr()` (`core/uverbs_cmd.c`) in the uapi tree. Before any driver code runs, the handler:

- checks that `start` and `hca_va` have the same offset within a page: the NIC's pages must line up with the CPU's;
- reserves the new MR's handle with `uobj_alloc(UVERBS_OBJECT_MR, …)`, which leaves a NULL slot in the XArray;
- checks the access flags in `ib_check_mr_access()`. Remote write or atomic access requires `IB_ACCESS_LOCAL_WRITE`, and on-demand paging requires device support;
- looks up and read-locks the PD with `uobj_get_obj_read(pd, UVERBS_OBJECT_PD, cmd.pd_handle, attrs)`.

Then it calls the driver: `pd->device->ops.reg_user_mr(pd, start, length, hca_va, access, NULL, &attrs->driver_udata)`. For mlx5 that is `mlx5_ib_reg_user_mr()` (row 9, `hw/mlx5/mr.c`). With `IB_ACCESS_ON_DEMAND` it takes the ODP path and pins nothing. Otherwise it pins, then builds the MR:

```c
	umem = ib_umem_get_va(&dev->ib_dev, start, length, access_flags);
	if (IS_ERR(umem))
		return ERR_CAST(umem);
	return create_real_mr(pd, umem, iova, access_flags, dmah);
```

### Pinning the pages (rows 10–12)

`ib_umem_get_va()` is an `ib_core` helper (`core/umem.c`), the same one that pins QP and CQ rings below. It is where registration spends most of its time, and the only step whose cost grows with the buffer size:

![Pinning a buffer that touches 7 pages: check, charge 7 pages to pinned_vm against RLIMIT_MEMLOCK, pin_user_pages_fast in batches of up to 512 pages with FOLL_LONGTERM and FOLL_WRITE, append the pages to a scatter-gather table merging physically contiguous runs, and DMA-map it; the 7 physical pages fall into 3 contiguous runs and become 3 segments](/assets/images/posts/rdma-3-control-plane/pin.svg)

The core of it, with the error paths cut:

```c
	lock_limit = rlimit(RLIMIT_MEMLOCK) >> PAGE_SHIFT;
	new_pinned = atomic64_add_return(npages, &mm->pinned_vm);
	if (new_pinned > lock_limit && !capable(CAP_IPC_LOCK))
		goto out;                               /* -ENOMEM */

	cur_base = addr & PAGE_MASK;
	/* gup_flags starts as FOLL_LONGTERM */
	if (umem->writable)
		gup_flags |= FOLL_WRITE;

	while (npages) {
		pinned = pin_user_pages_fast(cur_base,
				min_t(unsigned long, npages,
				      PAGE_SIZE / sizeof(struct page *)),
				gup_flags, page_list);
		cur_base += pinned * PAGE_SIZE;
		npages -= pinned;
		ret = sg_alloc_append_table_from_pages(&umem->sgt_append,
				page_list, pinned, 0, pinned << PAGE_SHIFT,
				ib_dma_max_seg_size(device), npages, GFP_KERNEL);
	}

	ret = ib_dma_map_sgtable_attrs(device, &umem->sgt_append.sgt,
				       dir, umem->dma_attrs);
```

What each part is for:

- **`RLIMIT_MEMLOCK` accounting.** Pinned pages can never be reclaimed, so they count against the process's lock limit through `mm->pinned_vm`. This is why RDMA programs need a large `ulimit -l`, and why containers running them are usually given `--ulimit memlock=-1`. `CAP_IPC_LOCK` bypasses the check. Whole pages are charged, even when the buffer covers only part of the first and last page.
- **`pin_user_pages_fast()`** faults the pages in if needed and takes a *pin*, not just a reference. The page's refcount is raised by `GUP_PIN_COUNTING_BIAS` (large folios keep a separate pin count), so the mm can tell with `folio_maybe_dma_pinned()` that a device is using it. Reclaim, migration and compaction leave such pages alone, and `fork()` copies them for the child instead of sharing them. The batch size is one page of `struct page *` pointers: 512 on a 64-bit kernel.
- **`FOLL_LONGTERM`** says the pin may last indefinitely. GUP first migrates the pages out of `ZONE_MOVABLE` and CMA areas, so a long-lived pin does not block memory hot-unplug or fragment CMA, and it refuses fs-dax mappings.
- **`FOLL_WRITE`** is added when the access flags allow writes (`ib_access_writable()`: local write, remote write, remote atomic or MW bind). It breaks copy-on-write now, so the pinned page is the process's own copy and a later write fault cannot swap in a different page behind the NIC's back.
- **`sg_alloc_append_table_from_pages()`** builds the scatter-gather table batch by batch and merges physically contiguous pages into one segment. A buffer in 2 MiB huge pages becomes a few large segments, not thousands of 4 KiB ones.
- **`ib_dma_map_sgtable_attrs()`** turns physical addresses into addresses the NIC can use: IOVAs if an IOMMU translates for the device, plain bus addresses if not. From now on the driver uses only these DMA addresses.

`ib_umem_release()` undoes it in reverse: DMA-unmap, unpin each run with `unpin_user_page_range_dirty_lock()` (marking pages dirty if the NIC could have written them), and subtract from `pinned_vm`.

### Choosing a page size (rows 14–15)

An mkey uses one page size for the whole region, and its translation table (the MTT) holds one 8-byte entry per page. Larger pages mean fewer entries to write and fewer translation-cache misses in the NIC. `mlx5_umem_mkc_find_best_pgsz()` (`mlx5_ib`) builds a bitmap of the page sizes the device supports. `ib_umem_find_best_pgsz()` (`ib_core`) then picks the largest one that fits the layout. A page size is only usable if every DMA segment boundary falls on a page boundary at the same offset as in `iova`. For an MR built from scattered 4 KiB pages the answer is 4 KiB. For a 1 GiB buffer in 2 MiB huge pages it is at least 2 MiB, which means 512 MTT entries instead of 262,144.

### The mkey: pool, firmware, UMR (rows 16–19)

`create_real_mr()` now needs an mkey, the NIC object behind `lkey` and `rkey`. It has two ways to get one:

- **The common path**, when the device can load this many translations with a UMR (`mlx5r_umr_can_load_pas()`): take a ready-made mkey and fill it in.
- **The slow path**, `reg_create()`, for very large regions or devices without that capability: one `CREATE_MKEY` command with the whole PAS list inline, under `dev->slow_path_mutex`.

The common path is a good example of the split this series is about: **a generic routine in `ib_core` that calls back into the driver.** Free mkeys are kept in *FRMR pools*, `core/frmr_pools.c`, added in 2026 by [`ce5df0b891ed`](https://github.com/torvalds/linux/commit/ce5df0b891ed) ("IB/core: Introduce FRMR pools"); mlx5 switched to them from its own MR cache in [`36680ef7bceb`](https://github.com/torvalds/linux/commit/36680ef7bceb). The pools sit in an rb-tree, one per key `{ access flags, ATS, num_dma_blocks, vendor bits }`. The driver registers three hooks when it registers the device:

```c
static const struct ib_frmr_pool_ops mlx5r_frmr_pool_ops = {
	.create_frmrs = mlx5r_create_mkeys,
	.destroy_frmrs = mlx5r_destroy_mkeys,
	.build_key = mlx5r_build_frmr_key,
};
```

![The mkey: ib_core keeps an rb-tree of FRMR pools keyed by access flags, ATS and number of DMA blocks; popping from an empty pool calls mlx5r_create_mkeys, which runs CREATE_MKEY with free=1 and umr_en=1; the popped mkey is then filled by mlx5r_umr_update_mr_pas with a UMR WQE carrying MTTs, PD, start, length and enable. Below: at runtime the HCA takes a key and virtual address, checks the mkey context's variant, PD, range and access, and indexes the MTT array to find the DMA address](/assets/images/posts/rdma-3-control-plane/mkey.svg)

1. **`ib_frmr_pool_pop()`** (`ib_core`, row 16) finds or creates the pool for this key and pops a free mkey handle. Only when the pool is empty does it call `pool_ops->create_frmrs`.
2. **`mlx5r_create_mkeys()`** (`mlx5_ib`, row 17) then builds an mkey context: `free = 1` (disabled), `umr_en = 1` (UMR may modify it), owned by the driver's internal UMR PD, with room for `num_dma_blocks` translations. **`mlx5_core_create_mkey()`** (`mlx5_core`, row 18) sends `CREATE_MKEY`. This is the only firmware command on the whole `reg_mr` path, and it does not run at all when the pool has a free mkey.
3. **`mlx5r_umr_update_mr_pas(mr, MLX5_IB_UPD_XLT_ENABLE, pdn)`** (`mlx5_ib`, row 19) fills the mkey in. It walks the umem with `rdma_umem_for_each_dma_block()`, writes one MTT entry per block into a DMA-able buffer, and posts it as one or more *UMR work requests* on a QP that the driver keeps for itself. The final one also sets the PD to the user's `pdn`, the start address (`iova`), the length, the access rights and the page size, and clears `free`. The driver then waits for the completion. The mkey is now live.

Once the mkey is live, the NIC uses it on every access: it checks the PD and the access rights, bounds-checks the address, and indexes the MTT, without asking the CPU.

### Back to user space, and back again

`set_mr_fields()` sets `lkey = rkey = mkey`. Back in `ib_uverbs_reg_mr()`, the handler fills in the generic `ib_mr` fields, increments `pd->usecnt` (so the PD cannot be freed under the MR), sets `uobj->object = mr`, drops the PD's read lock and commits the handle with `uobj_finalize_uobj_create()`. The response is `{ mr_handle, lkey, rkey }`.

`ibv_dereg_mr()` walks it back. `mlx5_ib_dereg_mr()` revokes a pooled mkey with a UMR (`free = 1`, PD switched back to the UMR PD) and pushes it back into its pool (`ib_frmr_pool_push()`). An mkey from `reg_create()`, or one whose revoke failed, is destroyed with `DESTROY_MKEY`. Then `ib_umem_release()` unpins the pages. A periodic aging job in `ib_core` destroys mkeys that have sat unused in a pool for too long.

## ibv_create_qp

![Call ladder of ibv_create_qp: libmlx5 sizes the WQEs, allocates the QP buffer and the doorbell record and picks a bfreg; libibverbs sends QP_CREATE by ioctl; the uverbs handler resolves the PD and CQs and calls ib_create_qp_user; ib_core allocates the mlx5_ib_qp and calls mlx5_ib_create_qp; create_user_qp maps the bfreg to a UAR page, pins the WQE buffer with ib_umem_get_attr_or_va, writes its pages into the command, pins the doorbell page, fills the QPC and sends CREATE_QP through mlx5_cmd_exec; ib_core adds security and tracking; uverbs commits the handle and returns the QP number](/assets/images/posts/rdma-3-control-plane/create-qp.svg)

### libmlx5 allocates the rings (rows 1–6)

For `reg_mr`, the provider added nothing. For a QP it does most of the planning, because the data path will run in libmlx5 and the rings must be laid out the way libmlx5 will use them. `create_qp()` (`providers/mlx5/verbs.c`):

- **sizes the queues.** A send WQE is built from 64-byte basic blocks, enough for the largest WR the caps allow. The SQ gets `max_send_wr × blocks per WR` blocks, rounded up to a power of two; the RQ stride follows from `max_recv_sge`.
- **allocates one buffer for both rings** (`mlx5_alloc_qp_buf()`): RQ at offset 0, SQ after it, page-aligned. It also allocates the `wrid` arrays that remember each WQE's `wr_id`, since the NIC never sees it.
- **allocates the doorbell record** (`mlx5_alloc_dbrec()`): 8 bytes, `{ receive counter, send counter }`, carved from a page of records shared by the context's QPs and CQs.
- **picks a doorbell register.** `mlx5_get_qp_uar()` gets a UAR page (allocated and `mmap()`ed on first use, see part 2) and passes its page ID with `MLX5_QP_FLAG_UAR_PAGE_INDEX`.

All of this goes into the driver part of the command, `struct mlx5_ib_create_qp { buf_addr, db_addr, sq_wqe_count, rq_wqe_count, rq_wqe_shift, flags, uidx, bfreg_index, … }`. `ibv_cmd_create_qp_ex()` then sends `ioctl(QP, QP_CREATE)` with the core attributes (PD, CQs, type, caps) plus this struct as `UHW_IN`.

### uverbs and ib_core (rows 7–10)

The `UVERBS_METHOD_QP_CREATE` handler (`core/uverbs_std_types_qp.c`) gets the PD and the CQs already looked up and read-locked by the attribute parser (part 2). It checks the QP type and caps and builds a `struct ib_qp_init_attr`. Then it calls `ib_create_qp_user()` → `create_qp()` in `core/verbs.c`. This is the common routine:

```c
	/* allocates sizeof(struct mlx5_ib_qp) */
	qp = rdma_zalloc_drv_obj_numa(dev, ib_qp);
	qp->device = dev;
	qp->pd = pd;
	qp->uobject = uobj;
	qp->qp_type = attr->qp_type;
	/* … send_cq, recv_cq, srq, event handler, restrack … */
	ret = dev->ops.create_qp(qp, attr,
				 uattrs ? &uattrs->driver_udata : NULL);
	/* … */
	ret = ib_create_qp_security(qp, dev);
	rdma_restrack_add(&qp->res);
```

`ib_core` allocates the driver's struct, sized by `INIT_RDMA_OBJ_SIZE(ib_qp, mlx5_ib_qp, ibqp)`, and fills in the generic fields. It hands the driver a QP to initialize, not a request to allocate one.

### mlx5_ib pins the rings and builds the QPC (rows 11–19)

`mlx5_ib_create_qp()` (`hw/mlx5/qp.c`) copies the driver struct from `udata`, validates the flags, and dispatches on type. A user RC QP goes to `create_user_qp()` → `_create_user_qp()`, which does three things to user memory:

1. **The doorbell register.** `MLX5_QP_FLAG_UAR_PAGE_INDEX` means `bfreg_index` already is a UAR page index, the one libmlx5 allocated. It goes into the QPC as `uar_page`.
2. **The WQE ring.** `ib_umem_get_attr_or_va(…, ucmd->buf_addr, buf_size, 0)` pins it, with the same `__ib_umem_get_va()` as an MR. The access is `0` because the NIC only reads this buffer: there is no `FOLL_WRITE` and the DMA direction is to the device. `mlx5_umem_find_best_quantized_pgoff()` picks the page size and offset the QPC can express. Then `mlx5_ib_populate_pas()` writes one DMA address per page straight into the `CREATE_QP` command, after the QPC.
3. **The doorbell record.** `mlx5_ib_db_map_user()` pins the *page* that contains `db_addr`, once per page. Other QPs and CQs whose records live on the same page share that umem through a refcounted entry on the ucontext's `db_page_list`. The QPC gets the record's DMA address as `dbr_addr`.

`create_user_qp()` then fills in the rest of the QP context. Service type `RC`, `pd` = the PD's `pdn`, `cqn_snd`/`cqn_rcv` = the CQ numbers, and the log sizes and strides of both rings. `mlx5_qpc_create_qp()` sets the opcode to `CREATE_QP` and runs `mlx5_cmd_exec()`. The firmware creates the QP in state `RST` and returns its number. The driver sets `ibqp.qp_num` and adds the QP to the device's table, so that asynchronous events can find it.

On the way out, `ib_core` registers the QP with the P_Key security code and the resource tracker. `ib_uverbs` takes references on the PD and CQs (`ib_qp_usecnt_inc()`), commits the handle, and copies out `qp_num` and the real caps. Back in libmlx5, `mlx5_store_qp()` records which QP owns that number (or user index), because completions name the QP that way. `map_uuar()` then points the QP at its doorbell register inside the mapped UAR page.

### What the QP looks like afterwards

![After create_qp: in process memory, the RQ and SQ rings, the 8-byte doorbell record, the MR's data buffer, all pinned, and the mmap()ed UAR page; in the HCA, the QP context holds the ring's page addresses (PAS), the doorbell record's DMA address, the PD, the CQ numbers and the UAR page, next to the CQ and the mkey whose MTTs point at the data buffer](/assets/images/posts/rdma-3-control-plane/qp-memory.svg)

This is the arrangement that lets part 1's phase 4 skip the kernel. To send, libmlx5 writes a WQE into the next SQ slot in its own memory and bumps `snd_db` in the doorbell record. Then it writes the start of the WQE to the BlueFlame register, or the whole WQE if it is small. The HCA already knows from the QPC where the ring and the record are, by DMA address, and where to put completions (CQ `0x5c`). The `lkey` in each SGE selects the mkey that translates the buffer's addresses. None of this needs the kernel any more: it built the arrangement, pinned it, and left.

## ibv_modify_qp, briefly

`ibv_modify_qp()` is another `write()` command, with one extra job in `ib_core` for RoCE:

![Call ladder of ibv_modify_qp: write MODIFY_QP; ib_uverbs_modify_qp converts the attributes and looks up the QP; ib_core's _ib_modify_qp resolves the source GID entry and, for RoCE, the destination MAC, then calls ops.modify_qp through the security layer; mlx5_ib checks the attribute mask with ib_modify_qp_is_ok, converts attributes into QPC fields, picks the opcode from optab and runs it through mlx5_core](/assets/images/posts/rdma-3-control-plane/modify-qp.svg)

- **`ib_core` resolves addresses.** On RoCE the NIC builds Ethernet frames itself, so it needs a destination MAC. During `RTR`, `_ib_modify_qp()` looks up the source GID entry for `sgid_index` (`rdma_fill_sgid_attr()`). Then `ib_resolve_eth_dmac()` turns `dgid` into a MAC: directly for multicast and IPv6 link-local GIDs, otherwise through a route and neighbour lookup (`rdma_addr_find_l2_eth_by_grh()`).
- **`ib_core` provides the rules; the driver applies them.** `ib_modify_qp_is_ok()` holds the specification's table of required and optional attributes per transition. `mlx5_ib_modify_qp()` calls it to reject a bad `attr_mask`.
- **`mlx5_ib` translates.** `__mlx5_ib_modify_qp()` turns attributes into QPC fields and picks the opcode from a `[current][next]` state table, `optab`.

| Transition | Firmware opcode | Main QPC fields set (`MLX5_SET(qpc/ads, …)`) |
| --- | --- | --- |
| RESET → INIT | `RST2INIT_QP` | `pkey_index`, `vhca_port_num`; `rre`/`rwe`/`rae` from `qp_access_flags` |
| INIT → RTR | `INIT2RTR_QP` | `remote_qpn`, `next_rcv_psn`, `mtu`, `log_rra_max`, `min_rnr_nak`; path: `rgid_rip`, `rmac_47_32`, `src_addr_index`, `hop_limit`, `udp_sport` |
| RTR → RTS | `RTR2RTS_QP` | `next_send_psn`, `retry_count`, `rnr_retry`, `log_sra_max`, path `ack_timeout` |

`mlx5_core_qp_modify()` wraps the QPC in the opcode's mailbox and runs `mlx5_cmd_exec()`. After the third command, the QP is ready to send.

## Who owns what

| Module | `ibv_reg_mr` | `ibv_create_qp` |
| --- | --- | --- |
| rdma-core | `mlx5_reg_mr` → `ibv_cmd_reg_mr`: `write(REG_MR)` | libmlx5 sizes and allocates the rings, doorbell record and UAR; `ioctl(QP_CREATE)` |
| `ib_uverbs` | `ib_uverbs_reg_mr`: offset and access checks, response | `QP_CREATE` handler: type, caps, init attrs, response |
| `ib_uverbs_support` | reserve the MR handle, read-lock the PD, commit | reserve the QP handle, read-lock PD and CQs, commit |
| `ib_core` | `ib_umem_get_va` (pin + DMA-map), page size, FRMR pools | `create_qp` (allocates `mlx5_ib_qp`), pins the rings, security, restrack |
| `mm` | `pin_user_pages_fast()` with `FOLL_LONGTERM` | the same, for the ring and the doorbell page |
| `mlx5_ib` | `mlx5_ib_reg_user_mr`, `create_real_mr`, `mlx5r_create_mkeys`, UMR | `mlx5_ib_create_qp`, `create_user_qp`: PAS, doorbell, QPC |
| `mlx5_core` | `mlx5_core_create_mkey`, on a pool miss | `mlx5_cmd_exec()` |
| firmware | `CREATE_MKEY` (pool miss only); runs the UMR WQE | `CREATE_QP` → `qpn`; then the three modify commands |

## References

1. Linux, `drivers/infiniband/core/`: `uverbs_cmd.c` (`ib_uverbs_reg_mr`, `ib_uverbs_modify_qp`), `uverbs_std_types_qp.c`, `verbs.c` (`create_qp`, `_ib_modify_qp`, `ib_modify_qp_is_ok`), `umem.c` (`__ib_umem_get_va`, `ib_umem_find_best_pgsz`), `frmr_pools.c`.
2. Linux, `drivers/infiniband/hw/mlx5/`: `mr.c` (`mlx5_ib_reg_user_mr`, `create_real_mr`, `mlx5r_create_mkeys`), `umr.c` (`mlx5r_umr_update_mr_pas`), `qp.c` (`mlx5_ib_create_qp`, `_create_user_qp`, `__mlx5_ib_modify_qp`), `qpc.c`, `doorbell.c`.
3. Linux, `drivers/net/ethernet/mellanox/mlx5/core/`: `cmd.c` (`mlx5_cmd_exec`, the command queue), `mr.c` (`mlx5_core_create_mkey`); `include/linux/mlx5/mlx5_ifc.h` for the command and context layouts.
4. Linux, `mm/gup.c` and the `FOLL_PIN`/`FOLL_LONGTERM` notes in `include/linux/mm_types.h`; kernel documentation, [pin_user_pages() and related calls](https://docs.kernel.org/core-api/pin_user_pages.html).
5. rdma-core, `libibverbs/verbs.c`, `libibverbs/cmd_mr.c`, `libibverbs/cmd_qp.c`, `libibverbs/memory.c` (fork handling) and `providers/mlx5/verbs.c` (`create_qp`, `mlx5_reg_mr`).
6. Mellanox (NVIDIA), *Mellanox Adapters Programmer's Reference Manual (PRM)*, for ConnectX-4 and ConnectX-4 Lx: the command interface, mkeys, UMR and QP contexts.
