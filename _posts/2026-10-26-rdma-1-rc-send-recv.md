---
layout: page
permalink: /blogs/os/rdma-1-rc-send-recv.html
title: "RDMA Deep Dive 1: An RC Send/Receive, Verb by Verb"
date: 2026-10-26 11:00:00 -0700
topic: Operating System and Kernel Bypassing
math: false
description: >-
  The first part of a three-part RDMA deep dive writes a minimal RC send/receive program and
  then asks what its first two verbs really touch. ibv_get_device_list() never scans PCI: it
  dumps the kernel's registered RDMA devices over netlink. mlx5 registers one device per
  function (PF, VF or SF), with exceptions for switchdev, multiport and RoCE LAG. Bonding the
  two PFs replaces both devices with one mlx5_bond device and kills every context that was
  open on them. ibv_open_device() creates state per open file, not per process: a ufile with
  its own handle table and a ucontext with its own doorbell pages, over a shared device. A
  forked child shares the parent's file, but its write() commands are refused.
---

# RDMA Deep Dive 1: An RC Send/Receive, Verb by Verb

*RDMA deep dive: **1. RC send/receive** · [2. uverbs, ucontext, uobject, umem](/blogs/os/rdma-2-uverbs.html) · [3. Control plane: reg_mr and create_qp](/blogs/os/rdma-3-control-plane.html)*

This series follows the verbs from `libibverbs` down into the kernel and the mlx5 firmware. This part writes the program, then looks at what its first two verbs touch: which devices `ibv_get_device_list()` returns, what a RoCE bond does to them, and how the kernel serves `ibv_open_device()` to many processes. [Part 2](/blogs/os/rdma-2-uverbs.html) covers the uverbs machinery behind `cmd_fd`. [Part 3](/blogs/os/rdma-3-control-plane.html) traces `ibv_reg_mr()` and `ibv_create_qp()` to their firmware commands.

Code references are to Linux at [`d135ab453a74`](https://github.com/torvalds/linux/tree/d135ab453a7400a34930bdda1ef4bdc8acfac80e) (v7.3-16755, the v7.4 merge window) and rdma-core at [`07ce3f5`](https://github.com/linux-rdma/rdma-core/tree/07ce3f50f6e2924e33f12a1fdfb0a610d2d07f82). Kernel paths without a prefix are under `drivers/infiniband/`; `mlx5/core/` is `drivers/net/ethernet/mellanox/mlx5/core/`.

## Table of Contents

- [The program](#the-program)
- [ibv_get_device_list(): what is a device?](#ibv_get_device_list-what-is-a-device)
  - [How the list is built](#how-the-list-is-built)
  - [From PCI functions to ib_devices](#from-pci-functions-to-ib_devices)
  - [RoCE LAG: two PFs, one device](#roce-lag-two-pfs-one-device)
- [ibv_open_device(): one device, many openers](#ibv_open_device-one-device-many-openers)
  - [Shared versus per open](#shared-versus-per-open)
  - [How the kernel tells openers apart](#how-the-kernel-tells-openers-apart)
  - [fork() and passing the fd](#fork-and-passing-the-fd)
- [Why posting and polling need no system call](#why-posting-and-polling-need-no-system-call)
- [References](#references)

## The program

Hosts A and B, one RC QP each, RoCE v2 on ConnectX (mlx5). A sends 4 KiB with a `SEND` into a buffer that B has posted.

![The verbs objects: ibv_open_device turns an ibv_device into an ibv_context; the context creates a PD and a CQ; the PD creates an MR, which covers the buffer, and a QP, whose SQ and RQ refer to the CQ](/assets/images/posts/rdma-1-rc-send-recv/objects.svg)

| Object | What it is |
| --- | --- |
| `ibv_context` | An open device. It owns `cmd_fd`, the file descriptor every setup verb goes through. |
| `ibv_pd` | Protection domain. A QP may only use memory regions of its own PD. |
| `ibv_mr` | A registered buffer. `lkey` names it in local work requests, `rkey` in remote ones. |
| `ibv_cq` | Completion queue: where finished work requests are reported. |
| `ibv_qp` | A send queue (SQ) and a receive queue (RQ). Its 24-bit `qp_num` is what the peer connects to. |

In the listing, outputs worth keeping are in the comments. `attr` is `struct ibv_qp_attr`, and the mask flags drop their `IBV_QP_` prefix. Error checks are left out.

```c
/* both hosts: open the device, allocate resources */
dev = ibv_get_device_list(&n)[0];     /* "mlx5_0" */
ctx = ibv_open_device(dev);           /* ctx->cmd_fd */
ibv_query_port(ctx, 1, &port);        /* port.active_mtu, .state */
ibv_query_gid(ctx, 1, gid_idx, &gid); /* RoCE v2: an IP address */
pd  = ibv_alloc_pd(ctx);
mr  = ibv_reg_mr(pd, buf, 4096, IBV_ACCESS_LOCAL_WRITE);
                                      /* mr->lkey, mr->rkey */
cq  = ibv_create_cq(ctx, 16, NULL, NULL, 0);  /* cq->cqe >= 16 */
qp  = ibv_create_qp(pd, &(struct ibv_qp_init_attr){
          .qp_type = IBV_QPT_RC, .send_cq = cq, .recv_cq = cq,
          .cap = { .max_send_wr = 16, .max_recv_wr = 16,
                   .max_send_sge = 1, .max_recv_sge = 1 } });
                                 /* qp->qp_num; .cap rounded up */

/* both: swap { qp_num, psn, gid } over TCP, then */
ibv_modify_qp(qp, &(attr){ .qp_state = IBV_QPS_INIT,
                  .pkey_index = 0, .port_num = 1,
                  .qp_access_flags = 0 },
              STATE | PKEY_INDEX | PORT | ACCESS_FLAGS);
ibv_modify_qp(qp, &(attr){ .qp_state = IBV_QPS_RTR,
                  .path_mtu = port.active_mtu,
                  .dest_qp_num = remote.qpn, .rq_psn = remote.psn,
                  .max_dest_rd_atomic = 1, .min_rnr_timer = 12,
                  .ah_attr = { .is_global = 1, .port_num = 1,
                               .grh = { .dgid = remote.gid,
                                        .sgid_index = gid_idx,
                                        .hop_limit = 1 } } },
              STATE | AV | PATH_MTU | DEST_QPN | RQ_PSN |
              MAX_DEST_RD_ATOMIC | MIN_RNR_TIMER);
ibv_modify_qp(qp, &(attr){ .qp_state = IBV_QPS_RTS,
                  .sq_psn = local.psn, .timeout = 14,
                  .retry_cnt = 7, .rnr_retry = 7,
                  .max_rd_atomic = 1 },
              STATE | SQ_PSN | TIMEOUT | RETRY_CNT | RNR_RETRY |
              MAX_QP_RD_ATOMIC);

/* data: B posts a receive and tells A, then A sends */
sge = (struct ibv_sge){ .addr = buf, .length = 4096,
                        .lkey = mr->lkey };
/* B */ ibv_post_recv(qp, &(struct ibv_recv_wr){ .wr_id = 1,
                          .sg_list = &sge, .num_sge = 1 }, &bad);
/* A */ ibv_post_send(qp, &(struct ibv_send_wr){ .wr_id = 2,
                          .opcode = IBV_WR_SEND,
                          .send_flags = IBV_SEND_SIGNALED,
                          .sg_list = &sge, .num_sge = 1 }, &bad);
while (ibv_poll_cq(cq, 1, &wc) == 0)
        ;                /* wc.status, .opcode, .wr_id, .byte_len */

/* teardown in reverse: destroy_qp, destroy_cq, dereg_mr,
   dealloc_pd, close_device */
```

![Sequence of the two hosts: both set up resources; A sends qpn, psn and gid to B and B replies with its own; both move their QPs through INIT and RTR; B posts its receive, goes to RTS and tells A it is ready; A posts the SEND, the packet reaches B's posted buffer, B's NIC returns an ACK; B gets an IBV_WC_RECV completion and A an IBV_WC_SEND completion](/assets/images/posts/rdma-1-rc-send-recv/rc-sequence.svg)

A few things the listing does not show:

- The connection values are the ones rdma-core's [`rc_pingpong.c`](https://github.com/linux-rdma/rdma-core/blob/07ce3f50f6e2924e33f12a1fdfb0a610d2d07f82/libibverbs/examples/rc_pingpong.c) uses. The PSNs cross over: each side's `rq_psn` is the peer's `sq_psn`.
- `RTR` is enough to receive and acknowledge. B needs `RTS` only if it also sends.
- If the `SEND` arrives before the receive is posted, B's NIC answers with an RNR NAK, and A retries after `min_rnr_timer` (12 is 0.64 ms). `rnr_retry = 7` means retry forever.
- `wr_id` never reaches the NIC. libmlx5 stores it in a per-QP array indexed by WQE slot and fills the WC from the WQE counter in the completion.

## ibv_get_device_list(): what is a device?

### How the list is built

`libibverbs` does not scan PCI. `ibverbs_get_device_list()` (`libibverbs/init.c`) asks the kernel over RDMA netlink:

1. **`RDMA_NLDEV_CMD_GET`, as a dump.** The kernel walks its registered `ib_device`s and returns the ones the caller's network namespace may see (`rdma_dev_access_netns()` in `core/device.c`). That is all of them in the default *shared* mode. In *exclusive* mode (`rdma system set netns exclusive`), each device belongs to one namespace, and `ib_uverbs_open()` makes the same check.
2. **`RDMA_NLDEV_CMD_GET_CHARDEV` for the `"uverbs"` client**, once per device. `ib_uverbs_get_nl_info()` answers with the character device (`uverbsN`), the ABI version and the `driver_id`.
3. **Provider matching.** `driver_id == RDMA_DRIVER_MLX5` selects libmlx5. Matching on the PCI modalias from sysfs is only a fallback, and so is scanning `/sys/class/infiniband_verbs/` on kernels without netlink support.

Every call does this again, then compares the result with the previous list. A device counts as the same entry only if both its char-dev name and its kernel index match. The index comes from `xa_alloc_cyclic()`, so it is not reused right away. A device that is unregistered and registered again, as with LAG below, is a new entry even if it ends up with the same name.

### From PCI functions to ib_devices

![PCI functions to RDMA devices: PF0, PF1, a VF and an SF each get an mlx5_core.rdma auxiliary device, which mlx5_ib turns into an ib_device mlx5_0 to mlx5_3 with char devices uverbs0 to uverbs3. Exceptions: VFs bound to vfio-pci, enable_rdma false, switchdev representors, multiport RoCE, RoCE LAG. Below: libibverbs enumerates with RDMA_NLDEV_CMD_GET, then GET_CHARDEV, then matches a provider by driver_id](/assets/images/posts/rdma-1-rc-send-recv/devices.svg)

`mlx5_core` binds every PCI function, PF or VF, and every sub-function (SF, a function on the auxiliary bus created with `devlink port add`). Each gets its own `mlx5_core_dev`, and on it `mlx5_core` creates *auxiliary devices*: `eth`, `rdma`, and others (`mlx5/core/dev.c`). `mlx5_ib` is the driver of the `rdma` one (`mlx5r_probe()`) and registers one `ib_device` for it.

So on a plain host the answer is yes: there is one RDMA device per PF, plus one per VF or SF whose function this kernel's `mlx5_core` owns. A dual-port card is two PFs, so it shows up as two single-port devices, not one device with two ports. The exceptions, all decided in `mlx5/core/dev.c`:

| Case | What you see |
| --- | --- |
| VF given to a VM with `vfio-pci` | Nothing on the host. The guest's own `mlx5_core` creates a device inside the VM. |
| devlink `enable_rdma` set to false | No `rdma` auxiliary device, so no RDMA device. |
| eswitch in switchdev mode | An `rdma-rep` auxiliary device instead: one IB device with one port per vport (representor). |
| multiport RoCE (affiliated ports) | The second PF's `multiport` auxiliary device joins the first PF's device as port 2: one device, two ports. |
| RoCE LAG | Both PFs' devices go away and one `mlx5_bond_N` appears (next section). |

Names come from `ib_register_device(…, "mlx5_%d")`, which takes the next free number, so `mlx5_0` is simply whichever function registered first. rdma-core also ships a udev rule (`rdma_rename NAME_FALLBACK`) that renames devices to stable names such as `rocep8s0f0` where it can. `dev_list[0]` is therefore arbitrary: real programs pick a device by name, node GUID or netdev.

### RoCE LAG: two PFs, one device

![RoCE LAG: before, PF0 and PF1 each have their own device, mlx5_0 and mlx5_1, open by a process. Enslaving both netdevs to bond0 makes mlx5_do_bond remove both rdma auxiliary devices, send CREATE_LAG, and add the rdma auxiliary device on PF0 only, which registers mlx5_bond_0 with one port whose state and GIDs come from bond0. Open contexts on the old devices get DEVICE_FATAL and -EIO](/assets/images/posts/rdma-1-rc-send-recv/lag.svg)

**When it happens.** `mlx5_core` watches netdev events. LAG starts when the two PF netdevs, and nothing else, are slaves of one bond in active-backup or a hash mode (802.3ad, balance-xor). Prerequisites are checked in `mlx5_lag_check_prereq()`: RoCE enabled on both, the same eswitch mode, and no VFs unless the eswitch is in switchdev mode.

**What `mlx5_do_bond()` does** (`mlx5/core/lag/lag.c`):

1. `mlx5_lag_remove_devices()`: sets `MLX5_PRIV_FLAGS_DISABLE_IB_ADEV` on both PFs and rescans, which deletes their `rdma` auxiliary devices. `mlx5_ib` unregisters `mlx5_0` and `mlx5_1`.
2. `mlx5_activate_lag()`: the `CREATE_LAG` firmware command joins the two ports.
3. Re-enables the `rdma` auxiliary device on the first PF only. `mlx5_ib_stage_ib_reg_init()` sees `mlx5_lag_is_active()` and registers the device as `mlx5_bond_%d`.

The bond device has one port. Its netdev is PF0's, but `mlx5_query_port_roce()` takes the port state and MTU from the bond master. The GID table follows the bond master too (`core/roce_gid_mgmt.c`), so `ibv_query_gid()` returns `bond0`'s addresses.

**What it does to a running program.** Unregistering `mlx5_0` makes uverbs *disassociate* every file open on it. `ib_uverbs_free_hw_resources()` walks the device's list of open files and calls `uverbs_destroy_ufile_hw(file, RDMA_REMOVE_DRIVER_REMOVE)` on each. QPs, CQs and MRs are destroyed in hardware and pinned memory is released. The async event fd delivers `IB_EVENT_DEVICE_FATAL`, and every later command fails with `-EIO`. The handles stay valid only for destroy and close. To continue, the program has to call `ibv_get_device_list()` again and open `mlx5_bond_0`. Removing the bond does the same in reverse.

**Which port a QP sends on.** With LAG active, each QP still transmits from one physical port at a time:

- **Queue affinity** (devices without a port-selection flow table). When a QP moves from RESET to INIT, `get_tx_affinity()` in `hw/mlx5/qp.c` gives it a TX port, round-robin from a counter in the ucontext. One process's QPs therefore alternate between the ports. If the address vector has a flow label, `_ib_modify_qp()` first asks the bond's own transmit hash which slave a matching UDP packet would leave on (`rdma_lag_get_ah_roce_slave()` in `core/lag.c`), and the QP gets that port.
- **Hash mode** (devices with a port-selection flow table). The firmware picks the port by hashing. If the device can also bypass the table per QP, the driver stops assigning affinity by default (`mlx5_ib_lag_should_assign_affinity()`).

When a slave loses link, `mlx5_core` recomputes which ports are usable and updates the LAG in firmware with `mlx5_modify_lag()`. The RDMA device and its QPs are untouched; that failover is the point of the feature.

## ibv_open_device(): one device, many openers

![Three processes opening uverbs0: process A opens it once, process B twice, and process C is a fork of A that inherits A's fd. That gives three struct files, each with its own ib_uverbs_file and mlx5_ib_ucontext; all three hang off one shared ib_uverbs_device and one shared ib_device. C shares A's file, and its write() commands fail with EACCES](/assets/images/posts/rdma-1-rc-send-recv/multi-open.svg)

### Shared versus per open

| Object | Scope | Holds |
| --- | --- | --- |
| `ib_device` (`mlx5_0`) | one per device | ports, GID table, `ib_device_ops`, FRMR pools, the driver's UMR QP |
| `ib_uverbs_device` | one per device | char dev `uverbs0`, uapi tree, refcount, `uverbs_file_list` of every open file |
| `struct file` | one per `open()` | `f_cred` of the opener; `private_data` → the ufile |
| `ib_uverbs_file` | one per `open()` | handle XArray, uobject list, `mmap()` tracking |
| `ib_ucontext` (`mlx5_ib_ucontext`) | one per `open()`, after `GET_CONTEXT` | UAR pages, transport domain, optional DEVX uid, the LAG round-robin counter |
| hardware namespaces | one per device | QP numbers, CQ numbers, mkeys: an rkey is a device-wide number |

### How the kernel tells openers apart

It does not identify processes at all. `ib_uverbs_open()` allocates a new `ib_uverbs_file` on every `open()` and stores it in `filp->private_data`; there is no table keyed by PID. Every `write()`, `ioctl()` and `mmap()` starts from the `struct file` it arrives on, and every handle is looked up in that file's XArray ([part 2](/blogs/os/rdma-2-uverbs.html#from-an-int-to-a-kernel-object)). Handle 1 in file #2 and handle 1 in file #3 are unrelated objects. A process that opens the device twice gets two contexts whose objects cannot be mixed: a PD handle from one is looked up in the other's table, where it is either missing or of the wrong type.

The shared `ib_uverbs_device` keeps every open file on `uverbs_file_list`, and it uses that list for one thing only: disassociating all of them when the device goes away, as in the LAG case.

The isolation does not stop at software:

- **Doorbells.** Each ucontext gets its own UAR pages (`ALLOC_UAR`), mapped into a process only through its own file. The QP context records which UAR page belongs to the QP.
- **Memory.** A QP can use only MRs of its own PD; the HCA checks the mkey's PD on every access. PDs are per-ucontext objects. rkeys are device-wide numbers and are meant to be used from the network; the PD and the access flags are what limit them.
- **DEVX.** When a ucontext was created with DEVX, its objects carry a firmware user ID, and the firmware checks commands against it.
- **Accounting.** Each ucontext charges one `RDMACG_RESOURCE_HCA_HANDLE` and each object one `RDMACG_RESOURCE_HCA_OBJECT` to the opener's rdma cgroup. Pinned pages count against the registering process's `mm->pinned_vm` ([part 3](/blogs/os/rdma-3-control-plane.html#pinning-the-pages-rows-1012)).

### fork() and passing the fd

Because state belongs to the open file, what matters is who holds the fd:

- **`fork()`**. The child inherits `cmd_fd` and with it the parent's ufile, ucontext and objects. Its `write()` commands fail with `EACCES`, though. `ib_uverbs_write()` calls `ib_safe_file_access()`, which requires `filp->f_cred == current_cred()`, and `fork()` gives the child its own `struct cred` (`copy_creds()` calls `prepare_creds()` unless `CLONE_THREAD`). The kernel logs "changed security contexts after opening file descriptor". `ioctl()` has no such check, so commands on the newer ABI work from the child, on the parent's objects. Threads share the cred and are not affected. The child's copies of the doorbell mappings are tracked by the same ufile (`rdma_umap_open()`), so a disassociate zaps them too.
- **Why the check exists.** It came in 2016 ([`e6bd18f57aad`](https://github.com/torvalds/linux/commit/e6bd18f57aad), "IB/security: Restrict use of the write() interface"). A uverbs `write()` carries pointers that the kernel writes its response through, and there are ways to make a different, more privileged context perform the `write()`, steering that response into kernel memory. The commit message names a structured `ioctl()` interface as the long-term fix. That is the ABI part 2 describes.
- **Sharing on purpose.** Send `cmd_fd` to another process over a UNIX socket (`SCM_RIGHTS`) and call `ibv_import_device(fd)` there. libmlx5's `import_context` rebuilds the user-space context from `QUERY_CONTEXT`, an `ioctl()`. Then `ibv_import_pd(ctx, handle)` and `ibv_import_mr(pd, handle)` wrap existing objects by handle number, which works only because both processes now use the same file and its handle table. The importer runs with different credentials, so the `write()` restriction applies to it as well.

## Why posting and polling need no system call

Everything above the `/* data */` comment in the listing is a system call on `cmd_fd`. `ibv_post_send()` is not. It writes the WQE into the SQ ring, which lives in the process's own memory, pinned by the kernel at `create_qp`. Then it updates the doorbell record and writes the doorbell register in the UAR page that the ucontext mapped. The NIC fetches the WQE and the payload by DMA and later writes a CQE into the CQ ring, also in process memory, where `ibv_poll_cq()` finds it:

![Control path versus data path: setup verbs go through write or ioctl on cmd_fd into ib_uverbs, ib_core, mlx5_ib and the mlx5_core command queue to the firmware; post_send writes a WQE into the SQ buffer and rings the doorbell by MMIO, the NIC DMA-reads the WQE and data and DMA-writes the CQE that poll_cq reads, without the kernel](/assets/images/posts/rdma-1-rc-send-recv/two-paths.svg)

How those rings, doorbell records and UAR pages get set up is the subject of [part 2](/blogs/os/rdma-2-uverbs.html) and [part 3](/blogs/os/rdma-3-control-plane.html).

## References

1. rdma-core, `libibverbs/init.c` (`ibverbs_get_device_list()`, provider matching), `libibverbs/ibdev_nl.c` (netlink enumeration), `libibverbs/device.c` (`ibv_import_device()`), `kernel-boot/rdma_rename.c` (stable names), and [`libibverbs/examples/rc_pingpong.c`](https://github.com/linux-rdma/rdma-core/blob/07ce3f50f6e2924e33f12a1fdfb0a610d2d07f82/libibverbs/examples/rc_pingpong.c).
2. Linux, `drivers/infiniband/core/`: `device.c` (`rdma_dev_access_netns()`, device indexes), `nldev.c`, `uverbs_main.c` (`ib_uverbs_open()`, `ib_uverbs_write()`, `ib_uverbs_get_nl_info()`, `ib_uverbs_free_hw_resources()`), `lag.c`, `roce_gid_mgmt.c`; `include/rdma/ib.h` (`ib_safe_file_access()`); `kernel/cred.c` (`copy_creds()`).
3. Linux, `drivers/net/ethernet/mellanox/mlx5/core/`: `dev.c` (auxiliary devices), `lag/lag.c` (`mlx5_do_bond()`); `drivers/infiniband/hw/mlx5/`: `main.c` (`mlx5_ib_stage_ib_reg_init()`, `mlx5_query_port_roce()`), `qp.c` (`get_tx_affinity()`).
4. rdma-core man pages: [ibv_get_device_list(3)](https://man7.org/linux/man-pages/man3/ibv_get_device_list.3.html), [ibv_open_device(3)](https://man7.org/linux/man-pages/man3/ibv_open_device.3.html), [ibv_modify_qp(3)](https://man7.org/linux/man-pages/man3/ibv_modify_qp.3.html), [rdma-system(8)](https://man7.org/linux/man-pages/man8/rdma-system.8.html).
5. InfiniBand Trade Association, *InfiniBand Architecture Specification, Volume 1*: ch. 9 (RC transport) and ch. 11 (verbs, QP state machine).
