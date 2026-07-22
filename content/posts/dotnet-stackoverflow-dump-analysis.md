---
title: 从 0xc00000fd 到具体方法：一次 .NET Stack Overflow 问题排查
date: 2026-07-22
draft: false
tags:
  - .NET
  - 故障排查
  - dump
  - StackOverflowException
categories:
  - 故障排查
description: 记录一次线上应用无响应问题的排查过程：从 Windows Event Log 的异常码，到 dotnet-dump、线程和符号文件，最终定位到插件中的无限递归。
---

线上有些问题很像“应用突然消失”：访问不到页面，应用日志里没有明显异常，服务却不断重启进程。遇到这种情况，Windows Event Log 往往比应用日志更早给出方向。

本文记录一次 .NET 应用 Stack Overflow 的排查过程。重点不在某个具体插件，而在于如何从一条异常码逐步缩小到具体的方法。

## 先从异常码确认方向

问题最初表现为应用无法访问，同时能看到大量应用异常退出，以及保活机制不断重启应用的日志。

Event Log 中有一条关键记录：

```text
Exception code: 0xc00000fd
```

在 Windows 中，`0xc00000fd` 通常对应栈溢出（`STATUS_STACK_OVERFLOW`）。

## 先拿到 dump，不要急着下结论

在 .NET 5 及更高版本中，可以参考 .NET 的 Stack Overflow 调试文档，让进程生成 dump。现场排查时，可以先让管理员权限启动相关服务，然后收集控制台输出；如果仍然需要更完整的现场信息，可以配置生成 mini dump。

拿到 dump 后使用 `dotnet-dump` 加载：

```powershell
dotnet-dump analyze .\dump.6012.dmp
```

一个容易踩到的坑是：加载完成后直接执行 `crashinfo`、`printexception` 或 `clrstack`，可能看不到有价值的信息：

```text
> crashinfo
ERROR: No crash info to display

> pe
There is no current managed exception on this thread
```

这并不表示 dump 没有问题。Stack Overflow 发生时，运行时可能来不及建立普通异常上下文；排查重点应该转向线程状态和栈帧本身。

## 从线程列表找到异常线程

先查看托管线程：

```text
> clrthreads
 DBG  ID  OSID  Count  Apt  Exception
   0    1  2604  -00001 MTA
   3    2  17ec  -00001 MTA (Finalizer)
   4    5  1638  -00001 MTA (Threadpool Worker)
  11   83  2c4c  -00001 MTA (Threadpool Worker) System.StackOverflowException
```

这里的关键信息是线程 11：它被标记为 `Threadpool Worker`，并且关联了 `System.StackOverflowException`。切换到这个线程后，再查看它的调用栈：

```text
> threads 11
> clrstack
OS Thread Id: 0x2c4c (11)
00000070084C5FF0  EhrUserManager.dll!Unknown
00000070084C5EE0  EhrUserManager.dll!Unknown
00000070084C5F20  EhrUserManager.dll!Unknown
00000070084C60F0  EhrUserManager.dll!Unknown
00000070084C61F0  EhrUserManager.dll!Unknown
00000070084C62F0  EhrUserManager.dll!Unknown
```

大量重复的同一模块栈帧，是无限递归的重要信号。不过此时方法名还是 `Unknown`，还不能直接判断是哪一行代码。

## 没有 PDB 时补齐符号

dump 里显示 `Unknown`，通常是因为缺少对应程序集的 PDB 符号文件。对于用户提供的插件，可以使用 ILSpy 反编译程序集并生成 PDB，然后把符号目录配置给 `dotnet-dump`：

```text
> setsymbolserver -directory C:\symbols\EhrUserManager
> loadsymbols
```

系统符号可以从 Microsoft Symbol Server 下载；自有插件则需要把生成的 PDB 放在对应目录，并确保 PDB 与程序集版本匹配。符号加载完成后再次查看调用栈：

```text
> clrstack
00000070084C5FF0  CIMD.ScanStep.Get(CIMD.ScanContext, System.String)
00000070084C60F0  CIMD.ScanStep.Get(CIMD.ScanContext, System.String)
00000070084C61F0  CIMD.ScanStep.Get(CIMD.ScanContext, System.String)
00000070084C62F0  CIMD.ScanStep.Get(CIMD.ScanContext, System.String)
```

这次可以看到 `CIMD.ScanStep.Get` 在栈上不断重复，基本可以确认该方法在某种输入下递归调用自己，且递归没有终止条件。

## 最终定位与经验

拿到插件源码后，代码中确实存在方法调用自身的路径。下一步是让插件作者在递归入口、关键参数和退出条件处增加日志，确认什么输入触发了不终止的递归。

这次问题的排查链路可以概括为：

1. 用 Event Log 的 `0xc00000fd` 判断栈溢出方向。
2. 生成并收集 dump。
3. 不依赖普通异常上下文，先用 `clrthreads` 找异常线程。
4. 切换线程查看 `clrstack`，通过重复栈帧识别递归。
5. 缺少符号时补齐 PDB，再把 `Unknown` 映射到具体方法。
6. 回到源码和输入数据，验证递归为何没有终止。

Stack Overflow 的重点往往不是“异常对象里有什么”，而是“哪个线程的栈正在重复增长”。当常规日志没有答案时，线程、调用栈和符号文件就是最可靠的线索。
