---
title: 避免挂载失败写入本地目录：给 Linux 挂载点加不可变保护
date: 2026-08-04
draft: false
tags:
  - Linux
  - 文件系统
  - 挂载
  - 运维
categories:
  - 运维实践
description: Linux 挂载失败时，程序可能把数据写入本地目录。本文用 chattr +i 保护挂载点，并说明原理与验证过程。
---

给程序配置一个目录来保存上传文件、备份或日志，是很常见的做法。目录可能位于外置硬盘、NFS 或其他网络存储上，例如：

```text
/srv/nas
```

但“目录存在”不代表“存储已经挂载”。如果网络中断、NFS 服务没有启动，或者开机时挂载单元失败，`/srv/nas` 仍然只是系统盘上的普通目录。程序并不知道这一点，照常读写后会发现读取到的状态与挂载的存储不同步。

这个问题的关键不是“如何让挂载更可靠”，而是要让挂载失败时的写入尽早失败。

## 挂载点为什么会接收写入

挂载点只是一个目录。挂载成功前，路径解析到的是父文件系统；挂载成功后，Linux VFS 才把这个目录覆盖成另一个文件系统的入口：

```text
挂载前：/srv/nas -> 根文件系统上的目录
挂载后：/srv/nas -> 外置磁盘或 NFS 文件系统
```

因此，挂载失败不会让路径自动消失，应用仍然可以对它执行 `open`、`mkdir` 和 `rename`。只要应用有权限，数据就会落到根文件系统。

## 用不可变属性让错误暴露出来

Linux 的 `chattr` 可以修改文件系统属性。对目录设置 `i`（immutable，不可变）属性后，目录内容不能被创建、删除、重命名或修改，即使调用者是 `root` 也不能直接绕过：

```bash
sudo chattr +i /srv/nas
```

这正好符合挂载点的保护需求：

1. 挂载还没有发生时，写入会收到 `Operation not permitted`，而不是悄悄占用系统盘；
2. 挂载成功后，路径进入外部文件系统，外部文件系统自己的权限和属性生效；
3. 外部存储掉线后，应用得到 I/O 错误，问题可以被监控发现。

`i` 属性作用在挂载点下面那个“被覆盖的目录”上。挂载成功后，这个目录暂时不可见，所以不会把它的权限带到新挂载的文件系统中。

## 一次完整的验证

下面以 `/srv/nas` 和 NFS 为例。生产环境请替换为自己的地址，并先在测试机验证。

### 1. 创建一个空挂载点

```bash
sudo install -d -o app -g app -m 0755 /srv/nas
```

挂载点最好保持为空。否则外部文件系统挂载后，目录中的原有文件会被隐藏；卸载后它们又会出现，容易造成“文件凭空消失”的误判。

### 2. 确认当前没有挂载

```bash
findmnt -T /srv/nas
mountpoint -q /srv/nas && echo '已经挂载，请先确认' || echo '当前未挂载'
```

如果 `findmnt` 显示的来源仍是根文件系统，说明此时可以继续设置属性。不要在没有确认的情况下对一个已挂载的目录运行 `chattr`，否则可能修改到外部文件系统根目录的属性。

### 3. 设置并检查不可变属性

```bash
sudo chattr +i /srv/nas
lsattr -d /srv/nas
```

输出中应包含 `i`，例如：

```text
----i---------e---- /srv/nas
```

现在即使使用 `root`，也不能在挂载失败的目录中创建文件：

```bash
sudo mkdir /srv/nas/should-fail
# mkdir: cannot create directory '/srv/nas/should-fail': Operation not permitted
```

### 4. 挂载外部存储并再次写入

```bash
sudo mount -t nfs4 192.168.23.156:/mnt/RAID/disk /srv/nas
findmnt -T /srv/nas
df -h /srv/nas
sudo -u app mkdir /srv/nas/created-on-nas
```

这时 `findmnt` 的 `SOURCE` 应该是 NFS 地址，`TARGET` 是 `/srv/nas`。最后一个 `mkdir` 成功，说明应用访问的是外部存储，而不是根文件系统上的目录。

验证时可以分别记录挂载前后的 `df -h` 和 `findmnt -T`，不要只根据目录能否访问来判断挂载是否成功。
