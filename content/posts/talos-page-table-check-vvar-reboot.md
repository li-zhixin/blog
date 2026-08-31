---
title: Talos 节点因 page_table_check BUG 重启：一次从误判到内核栈的调查
date: 2026-08-31
draft: false
tags:
  - Talos
  - Linux
  - 内核
  - Kubernetes
  - Docker
  - Testcontainers
  - 故障排查
categories:
  - 故障排查
description: 记录自动化容器测试触发 Talos 节点重启的完整调查过程，以及为什么最初没有发现 ramoops 中已经存在的内核崩溃栈。
---

自动化测试使用 Playwright、Testcontainers 和 Docker-in-Docker。测试运行一段时间后，`talos2` 会从 Kubernetes 中消失，iDRAC 报告 `System CPU Resetting`，对应的测试分片失败。

这次调查最终拿到了完整的内核 panic 栈。直接原因不是硬件 reset，也不是 `pid_max` 耗尽，而是 Linux 6.18 中 `PAGE_TABLE_CHECK` 对 time namespace 的 VVAR 页面错误记账，触发了 `BUG_ON()`。

## 最初看到的现象

高并发 testcontainers 场景下，监控 Pod 观察到：

```text
loadavg=90+
pids=6890
threads=12513
dstate_count=6
```

eBPF 还记录到一分钟内大量进程和网络命名空间操作：

```text
fork_parent[runc]:            1597
fork_parent[containerd-shim]: 1209
fork_parent[dockerd]:          779
netns operations:             1303
unshare:                        38
```

同时出现两类 D-state 栈。一类在网络命名空间清理路径：

```text
rcu_barrier
netdev_run_todo
default_device_exit_batch
ops_undo_list
cleanup_net
process_scheduled_works
worker_thread
```

另一类在容器文件清理路径：

```text
folio_wait_bit_common
folio_wait_writeback
truncate_inode_partial_folio
truncate_inode_pages_range
xfs_fs_evict_inode
evict
do_unlinkat
__x64_sys_unlinkat
```

这些现象很容易让人得出一个合理但不完整的结论：高并发容器创建、删除导致 netns、workqueue、XFS 和 NVMe 同时拥塞，最后触发 hard lockup。

这个结论描述了重启前的压力现场，但不是最终的重启原因。

## 第一次为什么没有找到崩溃栈

我们已经构建并部署了包含 ramoops 的定制内核：

```text
CONFIG_PAGE_TABLE_CHECK=y
CONFIG_PAGE_TABLE_CHECK_ENFORCED=y
CONFIG_PSTORE=y
CONFIG_PSTORE_RAM=y
CONFIG_PSTORE_CONSOLE=y
CONFIG_PSTORE_FTRACE=y
CONFIG_PSTORE_PMSG=y
```

启动参数也包含：

```text
memmap=4M$0x807fc00000
ramoops.mem_address=0x807fc00000
ramoops.mem_size=0x400000
ramoops.record_size=0x20000
ramoops.console_size=0x100000
ramoops.ftrace_size=0x40000
ramoops.pmsg_size=0x40000
```

但在 observer Pod 中执行 `ls -la /sys/fs/pstore` 得到的是空目录，于是我们误以为这次 reset 没有走 Linux panic 路径。

后来检查 mount namespace 才发现，`/sys/fs/pstore` 只是一个普通的 sysfs 目录，并没有挂载 `pstorefs`：

```text
/sys/fs/pstore is not a mountpoint
stat -f -c "%T" /sys/fs/pstore
sysfs
```

补充挂载后：

```bash
mount -t pstore pstore /sys/fs/pstore
```

立即出现：

```text
console-ramoops-0
dmesg-ramoops-0
dmesg-ramoops-1
```

因此“pstore 为空”并不代表没有崩溃记录。这里有两个独立问题：

1. ramoops 后端确实注册成功；
2. observer Pod 没有把 pstore 文件系统挂载出来。

## ramoops 中的决定性证据

`dmesg-ramoops-0` 保存了第一个 Oops：

```text
kernel BUG at mm/page_table_check.c:143!
Oops: invalid opcode: 0000 [#1]
CPU: 114 UID: 0 PID: 234883 Comm: ForkJoinPool.co
RIP: 0010:__page_table_check_zero+0xfb/0x130
```

关键调用栈：

```text
__page_table_check_zero+0xfb/0x130
__free_frozen_pages+0x52f/0x650
free_time_ns+0x85/0xc0
free_nsproxy+0x7f/0x130
do_exit+0x2ef/0xa30
do_group_exit+0x77/0x90
get_signal+0x702/0x790
arch_do_signal_or_restart+0x8f/0x280
exit_to_user_mode_loop+0x7c/0xf0
do_syscall_64+0x10d/0x1d0
entry_SYSCALL_64_after_hwframe+0x76/0x7e
```

第 143 行是：

```c
BUG_ON(atomic_read(&ptc->file_map_count));
```

含义是：内核准备释放一个页面时，发现 Page Table Check 仍认为该页面存在文件映射，页面映射计数没有归零，于是主动触发 BUG。

`dmesg-ramoops-1` 还保存了 panic 阶段的 NMI 信息：

```text
nmi_cpu_backtrace
nmi_trigger_cpumask_backtrace
sys_info
vpanic
panic
oops_end
do_trap
handle_invalid_op
exc_invalid_op
asm_exc_invalid_op
__page_table_check_zero
```

最后明确写着：

```text
Rebooting in 10 seconds..
```

这与运行时参数一致：

```text
panic_on_oops=1
panic=10
```

完整链路是：

```text
内核 BUG
  -> Oops
  -> panic_on_oops=1
  -> panic
  -> 10 秒后自动重启
```

因此 iDRAC 的 `System CPU Resetting` 在这次事件中是 Linux panic 重启后的管理控制器描述，而不是直接证据表明 BMC 主动复位。

## BUG 为什么会被容器测试触发

Linux 6.18 的 vDSO time namespace 使用特殊的 `[vvar]` 映射。该映射通过 PFN-mapped special PTE 安装，但 x86 上的 Page Table Check 仍把它当作普通用户可访问页面计数。

大部分这类页面不会被释放，所以错误记账长期不显眼。time namespace 的 VVAR 页面不同：它是通过 `alloc_page()` 分配的真实页面，最后一个 time namespace 使用者退出时会在 `free_time_ns()` 中释放。

于是出现：

```text
special PTE 被错误计数
  -> time namespace 退出
  -> free_time_ns() 释放 VVAR 页面
  -> file_map_count 仍非零
  -> __page_table_check_zero() BUG
```

Testcontainers 的高频容器创建和销毁会大量经过 runc、containerd-shim 和 Docker init 的 namespace 生命周期，正好放大了这个问题。fork、netns、XFS 和 workqueue 压力是触发场景和伴随症状；真正让内核主动重启的是 VVAR 页面计数错误。

这不是本地猜测。Talos 已有完全匹配的问题和修复：

- [Talos issue #13496](https://github.com/siderolabs/talos/issues/13496)
- [pkgs PR #1578](https://github.com/siderolabs/pkgs/pull/1578)
- 修复提交：`09cb04e048191a92de49261be6291595eda0ffda`

修复的核心是：Page Table Check 在 set/clear 路径都跳过 `pte_special()`，不再追踪这些 PFN-mapped special PTE。

## 调查过程中的误区

### 把高负载栈当成 reset 栈

`runc` 等待 XFS writeback、`kworker` 卡在 `cleanup_net` 都是真实栈，但它们只说明系统在重启前已经承受很大压力。没有 panic 日志时，不能把最后看到的栈直接称为重启调用栈。

### 把 pstore 目录为空当成没有 pstore 记录

必须同时确认三件事：后端是否注册、pstorefs 是否挂载、目录中是否有文件。只看一个路径是不够的。

### 先相信 iDRAC 的描述，再回头找 Linux 证据

`System CPU Resetting` 描述的是管理控制器看到的 reset 结果。只有把 iDRAC 时间、Linux boot ID、pstore 和内核日志对齐，才能判断 reset 是 Linux panic 还是外部复位。

## 以后遇到类似问题的检查顺序

```text
1. 对齐 iDRAC 时间、boot_id 和 uptime
2. 确认 pstore 后端是否注册
3. 确认 pstorefs 是否真的挂载
4. 读取 dmesg-ramoops 和 console-ramoops
5. 再用 eBPF 做压力路径和前置事件关联
```

这次最重要的经验不是“多加几个监控指标”，而是要先验证监控链本身是否可读。内核已经留下了决定性证据，但由于 pstorefs 没有挂载，我们一开始把一次明确的内核 BUG 误认为了没有调用栈的突然 reset。

## 当前结论

本次问题可以写成一条完整链路：

```text
高并发 Testcontainers 生命周期
  -> time namespace 大量创建/销毁
  -> Linux 6.18 PAGE_TABLE_CHECK 错误追踪 VVAR special PTE
  -> free_time_ns() 释放页面时 file_map_count 非零
  -> mm/page_table_check.c:143 BUG_ON()
  -> panic_on_oops=1
  -> panic=10
  -> Talos 重启
  -> 测试分片以 Unknown/255 失败
```
