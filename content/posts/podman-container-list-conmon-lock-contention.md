---
title: 容器化自动化测试中的 Podman 锁竞争排查
date: 2026-08-27
draft: false
tags:
  - Podman
  - Docker
  - Testcontainers
  - 并发
  - 性能排查
categories:
  - 故障排查
description: 通过 Docker 与 Podman 对照实验及 Podman 5.8.2 源码，分析 Docker 兼容 API 在容器停止期间的长尾阻塞路径。
---

我们的端到端测试是一个由 Playwright、Testcontainers 和容器运行时组成的多层工作流。

## 测试工作流

每个 Playwright worker 通过 Testcontainers 管理一组相互隔离的容器和 bridge network。启动前会清理残留资源，测试期间按需启动浏览器、应用和数据库，结束后再停止并删除容器和网络。

```text
清理残留资源
  -> 创建 network 和容器
  -> 执行浏览器测试
  -> stop/remove 容器
  -> remove network
```

这些操作都由 Testcontainers 通过 Docker 兼容 API 完成，后端运行时分别是 Docker 或 Podman。多个 worker 的 teardown 会让 stop、remove、network remove 与容器状态查询交错执行，这正是本次调查关注的并发场景。

## 实验条件

为了避免大型业务镜像的启动成本和业务行为干扰测试，只使用了一个小镜像：

- 工作负载：`busybox:1.36.1`
- Podman：`quay.io/podman/stable:v5.8.2-immutable`
- Docker：`docker:dind`，Docker Engine 29.7.2
- 每轮 24 个并发操作，共 5 轮
- 两个运行时均运行在 privileged 容器内，状态目录使用 4 GiB tmpfs

每个并发操作创建 bridge network，创建并启动一个 busybox 容器，然后执行 stop、remove 和 network remove。teardown 阶段同时发起 24 个 `GET /containers/json?all=1` 请求，并持续采样 containers-list、`/info` 和 `/_ping`。

## 完整 teardown：慢的不只是 network remove

所有时间单位均为毫秒。下面是 120 个样本的汇总：

| 操作 | Podman p50/p95/max | Docker p50/p95/max |
|---|---:|---:|
| contention containers-list | 4062 / 4220 / 4234 | 364 / 415 / 416 |
| container stop | 2235 / 4042 / 4422 | 1833 / 2650 / 2887 |
| container remove | 9.1 / 526 / 2224 | 5.3 / 8.7 / 15.6 |
| network remove | 1652 / 3690 / 3997 | 3066 / 4546 / 4834 |
| sampled `/info` max | 4221 | 42 |
| sampled `/_ping` p99 | 1.4 | 1.8 |

更有区分度的是三个 API 的表现：

- Podman 的 containers-list 和 `/info` 都接近 4 秒；
- `/_ping` 始终在毫秒级响应；
- Docker 的 containers-list 最慢也只有 416 ms，即使 Docker network remove 还会持续数秒。

因此，这不是整个 HTTP 服务失去响应，而是需要读取或同步容器状态的路径被阻塞了。

## Podman 源码中的锁路径

Podman 的 Docker 兼容接口 `/containers/json` 不只是读一个内存快照。Podman 5.8.2 的 `ListContainers` 会逐个把 libpod container 转成 Docker API 响应：

```text
ListContainers
  -> LibpodToContainer
    -> Container.State()
    -> Container.Inspect()
```

这里的 SHM lock 不是一把覆盖所有容器的全局锁。每个 `Container` 对象持有自己的 `lock.Locker`，对应共享内存中的一个锁 ID；底层由支持跨进程共享的 robust pthread mutex 实现。因此，不同 Podman API 请求甚至不同 Podman 进程，只要操作的是同一个容器，就会竞争同一把锁；操作不同容器则不会因为这把容器锁直接互斥。

锁保护的是单个容器的可变状态以及状态同步过程。`Container.State()` 和 `Container.Inspect()` 都会先获取该容器的锁，调用 `syncContainer()` 刷新状态，读取完成后才释放：

```text
container A SHM lock
  -> syncContainer()
  -> 读取 container A state/inspect data
  -> unlock
```

再看 stop 路径：

```text
StopWithArgs
  -> stopInternal
    -> 调用 OCI runtime 停止容器时暂时释放锁
    -> 重新获取容器 SHM lock
    -> waitForConmonToExitAndSave
      -> waitForExitFileAndSync
```

`StopWithArgs()` 进入时先获取容器锁。Podman 在调用 OCI runtime 停止容器时会暂时释放它，避免整个 stop timeout 都阻塞其他命令；但 OCI runtime 调用返回后，代码会重新获取同一把锁。随后执行 `waitForConmonToExitAndSave()`，并可能在 `waitForExitFileAndSync()` 中等待最多 5 秒。因为外层 `StopWithArgs()` 要到整个 stop 流程返回时才执行 `defer Unlock()`，所以这段 conmon/exit-file 等待仍处于持锁范围内。

于是，同一容器上的 stop 和状态查询会形成下面的等待关系：

```text
stop goroutine 持有 container SHM lock
    -> 等待 conmon/exit file
    -> containers-list 调用 State()/Inspect()
    -> 等待同一把 SHM lock
```

`ListContainers` 又是在一个 `for` 循环中逐个调用 `LibpodToContainer`。只要其中一个容器正在 stop 并持有自己的 SHM lock，转换就会停在这个容器上，后续容器也无法继续处理，整个 `/containers/json` 响应只能等待。这与完整 teardown 实验的时间特征吻合：container stop、containers-list 和 `/info` 都出现约四秒尾延迟，而 `/_ping` 不访问这些容器锁，因此不受影响。

相关源码位于 Podman v5.8.2：

- [`pkg/api/handlers/compat/containers.go`](https://github.com/containers/podman/blob/v5.8.2/pkg/api/handlers/compat/containers.go)：`ListContainers`、`LibpodToContainer`
- [`libpod/container.go`](https://github.com/containers/podman/blob/v5.8.2/libpod/container.go)：`Container.State`
- [`libpod/container_inspect.go`](https://github.com/containers/podman/blob/v5.8.2/libpod/container_inspect.go)：`Container.Inspect`
- [`libpod/container_internal.go`](https://github.com/containers/podman/blob/v5.8.2/libpod/container_internal.go)：`stopInternal`、`waitForConmonToExitAndSave`、`waitForExitFileAndSync`
- [`libpod/lock/shm_lock_manager_linux.go`](https://github.com/containers/podman/blob/v5.8.2/libpod/lock/shm_lock_manager_linux.go)：容器锁 ID 与共享内存锁的映射
- [`libpod/lock/shm/shm_lock.go`](https://github.com/containers/podman/blob/v5.8.2/libpod/lock/shm/shm_lock.go)：跨进程 robust pthread mutex

Docker 的情况不同。dockerd 的 containers-list 从内存中的 MVCC view 读取，不会在枚举每个容器时走 Podman 这条 `State()/Inspect()/SHM lock` 路径。因此 Docker 也可能慢于 network remove，但容器列表没有被同一类容器生命周期锁耦合。

## 结论

Podman 5.8.2 的 containers-list 会逐个调用 `State()` 和 `Inspect()` 读取容器实时状态，并可能与 stop 后的 conmon 清理竞争容器 SHM 锁。Docker 的 containers-list 主要读取 dockerd 的内存 MVCC view，不需要在枚举时逐个获取容器生命周期使用的锁。因此，即使 Docker 的 network remove 较慢，containers-list 仍能快速返回。
