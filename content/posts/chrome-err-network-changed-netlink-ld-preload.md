---
title: 用 LD_PRELOAD 解决容器环境中的 Chrome ERR_NETWORK_CHANGED
date: 2026-07-23
draft: false
tags:
  - Chrome
  - Linux
  - Netlink
  - Kubernetes
  - 自动化测试
categories:
  - 工程实践
description: Chrome 会监听 Linux 网络变化；在频繁创建容器的 CI 环境中，这种正确行为反而会中断无关请求。本文介绍如何通过 LD_PRELOAD 拦截 NETLINK_ROUTE 套接字。
---

运行 Playwright 自动化测试时，我们遇到过一种很难稳定复现的失败：页面正在正常加载，Chrome 却突然报错：

```text
net::ERR_NETWORK_CHANGED
```

它看起来像目标服务断网，但实际检查后发现：

- 被访问的服务没有重启
- Pod 本身的网络仍然可用
- 同一时刻，Pod 内的容器引擎正在创建或销毁其他容器
- 重试通常又能成功

真正变化的不是 Chrome 正在使用的网络，而是同一网络命名空间中的虚拟网卡。

## 为什么启动容器会影响 Chrome

在服务器运行 Docker 时，每次创建或销毁容器，都可能伴随这些操作：

- 创建或删除 veth 设备
- 给虚拟网卡增加或移除 IP 地址
- 更新链路状态和路由信息

这些变化会通过 Linux Netlink 机制通知用户空间。Chrome 恰好也是这些通知的订阅者。

Chromium 在 Linux 上使用 [`AddressTrackerLinux`](https://source.chromium.org/chromium/chromium/src/+/main:net/base/address_tracker_linux.cc) 跟踪网络状态。初始化时，它会创建一个路由 Netlink 套接字：

```cpp
netlink_fd_.reset(socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE));
```

随后订阅多组网络事件：

```cpp
addr.nl_groups =
    RTMGRP_IPV4_IFADDR |
    RTMGRP_IPV6_IFADDR |
    RTMGRP_NOTIFY |
    RTMGRP_LINK;
```

因此，同一网络命名空间里任何网卡的地址或链路发生变化，都可能被 Chrome 观察到。事件再经由 [`NetworkChangeNotifierLinux`](https://source.chromium.org/chromium/chromium/src/+/main:net/base/network_change_notifier_linux.cc) 向网络栈传播，最终让正在进行的请求以 `ERR_NETWORK_CHANGED` 结束。

整个链路可以简化成：

```text
创建或销毁容器
      ↓
虚拟网卡、IP 或链路状态变化
      ↓
内核发送 NETLINK_ROUTE 消息
      ↓
Chrome 的 AddressTrackerLinux 收到变化
      ↓
NetworkChangeNotifier 通知网络栈
      ↓
正在进行的请求被中断
```

Chrome 的行为对桌面用户是合理的。例如从 Wi-Fi 切换到有线网络后，旧连接确实可能已经失效。但在 CI 中，其他容器产生的虚拟网卡变化通常与浏览器访问的目标毫无关系。

## 为什么禁用 IPv6 只能缓解

网络上常见的建议是禁用 IPv6。它有时有效，是因为减少了 IPv6 地址变更事件，但 Chrome 监听的不只有 IPv6：

```text
RTMGRP_IPV4_IFADDR
RTMGRP_IPV6_IFADDR
RTMGRP_NOTIFY
RTMGRP_LINK
```

容器创建时产生的 IPv4 地址和链路事件仍然存在。因此，禁用 IPv6 只能减少触发次数，不能从机制上切断这条通知链路。

## 思路：让 Netlink 监听初始化失败

问题的关键不是 Chrome 能否访问网络，而是它是否需要感知这个环境中的动态网络变化。

查看 Chromium 的 `AddressTrackerLinux::Init()` 可以发现，当 `NETLINK_ROUTE` 套接字创建失败时，它会执行下面的分支：

```cpp
netlink_fd_.reset(socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE));
if (!netlink_fd_.is_valid()) {
    PLOG(ERROR) << "Could not create NETLINK socket";
    AbortAndForceOnline();
    return;
}
```

`AbortAndForceOnline()` 会停止 Netlink 监听，并把当前连接类型设置为 `CONNECTION_UNKNOWN`。Chrome 仍然可以创建普通的 TCP、UDP 和 Unix Domain Socket，只是不再接收这一路网络变化通知。

所以可以把目标收缩为：

> 只让 `socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)` 失败，其他 `socket()` 调用保持原样。

## 用 LD_PRELOAD 替换 socket()

Linux 动态链接器支持通过 `LD_PRELOAD` 预先加载共享库。如果共享库导出了与 libc 相同的 `socket()` 符号，动态链接器会优先使用这个版本。

实现只需要 31 行 C 代码：

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <sys/socket.h>
#include <linux/netlink.h>
#include <dlfcn.h>
#include <errno.h>

static int (*real_socket)(int, int, int) = NULL;

int socket(int domain, int type, int protocol) {
    if (domain == AF_NETLINK &&
        (type & SOCK_RAW) == SOCK_RAW &&
        protocol == NETLINK_ROUTE) {
        fprintf(stderr,
                "Intercepted and blocked socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE) call.\n");
        errno = EPERM;
        return -1;
    }

    if (!real_socket) {
        real_socket = dlsym(RTLD_NEXT, "socket");
        if (!real_socket) {
            fprintf(stderr,
                    "LD_PRELOAD critical error: could not find real socket() function.\n");
            errno = EFAULT;
            return -1;
        }
    }

    return real_socket(domain, type, protocol);
}
```

这段代码分成两条路径。

第一条路径只匹配目标调用：

```c
domain 是 AF_NETLINK
type 判断命中 SOCK_RAW
protocol 是 NETLINK_ROUTE
```

匹配后返回 `-1`，并把 `errno` 设置为 `EPERM`，让调用方得到一次正常的系统调用失败。

第二条路径处理所有未命中的套接字。`dlsym(RTLD_NEXT, "socket")` 会跳过当前共享库，继续寻找下一个 `socket()` 实现，也就是 libc 中的真实函数。这样浏览器的正常网络请求不会被代理或修改。不过，这个拦截条件对整个进程生效，不只针对 Chromium 的 `AddressTrackerLinux`。

## 编译与接入

把代码编译成位置无关的共享库：

```bash
gcc -shared -fPIC -o fake_netlink.so fake_netlink.c -ldl
```

启动 Chrome 前设置环境变量：

```bash
export LD_PRELOAD=/opt/netlink/fake_netlink.so
```

如果使用 Playwright，可以只把变量传给浏览器进程：

```ts
import path from 'node:path';

const browser = await chromium.launch({
  env: {
    ...process.env,
    LD_PRELOAD: path.resolve('netlink/fake_netlink.so'),
  },
});
```

这比在整个测试容器中全局导出 `LD_PRELOAD` 更容易控制影响范围。Chrome 的子进程会继承环境变量，因此它的多进程架构不需要额外处理。

## 如何确认拦截生效

启动 Chrome 后，标准错误中应该出现：

```text
Intercepted and blocked socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE) call.
```

也可以使用 `strace` 观察原始行为：

```bash
strace -f -e trace=socket,bind chromium 2>&1 | grep NETLINK_ROUTE
```

未注入共享库时，可以看到 Chrome 成功创建 `NETLINK_ROUTE` 套接字。注入后，日志应显示目标调用被拦截，同时普通页面仍然可以访问。

## 结语

沿着 `ERR_NETWORK_CHANGED` 追到 Chromium 的 Netlink 监听后，我们用 `LD_PRELOAD` 屏蔽无关的网络变化，让容器环境中的自动化测试恢复稳定。
