---
title: 线程池饥饿排查：从 ThreadPool 指标追到 SQLite_BUSY
date: 2026-07-22
draft: false
tags:
  - .NET
  - 线程池
  - SQLite
  - 性能排查
categories:
  - 性能优化
description: 通过 dotnet-counters 和 dotnet-stack 排查线程池饥饿，并分析 SQLite WAL 合并、连接关闭与 SQLITE_BUSY 之间的关系。
---

应用变慢时，第一反应很容易是“线程不够”或“CPU 不够”。但在高并发 Web 应用中，更常见的情况是线程池线程都被阻塞了：线程数量不断增加，工作项排队等待，新的请求迟迟得不到执行。

这就是线程池饥饿（Thread Pool Starvation）。本文用一次 SQLite 相关的现场问题说明如何观察它，以及为什么简单增加线程数往往会让问题更糟。

## 什么是线程池饥饿

当线程池没有可用线程处理新的工作项时，就发生了线程池饥饿。应用通常表现为响应变慢、请求超时，但 CPU 利用率未必很高。

常见诱因包括：

- 在异步方法上调用 `.Result` 或 `.Wait()`；
- 使用 `Thread.Sleep`；
- 长时间运行的定时任务或后台任务占用线程池线程；
- 同步阻塞 I/O；
- 多个线程竞争同一把锁；
- 数据库调用长时间等待锁释放。

尤其要注意，业务代码不需要“完全同步”才会产生饥饿。只要异步链路中夹杂了阻塞调用，线程池就可能在压力下逐渐耗尽可用线程。

## 用 dotnet-counters 观察现场

先对目标进程做压测，同时运行：

```powershell
dotnet-counters monitor -p 12256
```

重点观察 `[System.Runtime]` 下的几个指标：

```text
ThreadPool Queue Length       75
ThreadPool Thread Count       38
ThreadPool Completed Work Item Count  128 / sec
```

如果 `ThreadPool Queue Length` 持续增长，说明工作项已经开始排队；如果 `ThreadPool Thread Count` 长时间明显高于 CPU 核心数，同时队列仍然无法消化，这是线程池饥饿的强烈信号。现场案例中线程数长期达到 CPU 核心数约 3 倍，仍有大量工作项等待。

这个判断应该结合趋势和请求延迟一起看，不能把“线程数超过核心数”单独当成故障证明。线程池在遇到阻塞时主动注入线程本身就是一种正常的补救机制，问题在于注入的线程也被同样的阻塞点卡住了。

## 用 dotnet-stack 看谁在阻塞

线上通常无法附加 debugger，这时可以使用 `dotnet-stack` 获取托管线程的调用栈。现场看到的一类关键栈如下：

```text
Thread (0x77B8):
  [Native Frames]
  SQLitePCLRaw.provider.e_sqlite3!
    SQLite3Provider_e_sqlite3.sqlite3_prepare_v2()
  Microsoft.Data.Sqlite!
    SqliteCommand.PrepareAndEnumerateStatements()
  Microsoft.Data.Sqlite!
    SqliteCommand.GetStatements()
  Microsoft.Data.Sqlite!
    SqliteDataReader.NextResult()
  Microsoft.Data.Sqlite!
    SqliteCommand.ExecuteReader()
  Microsoft.Data.Sqlite!
    SqliteCommand.ExecuteNonQuery()
```

这个栈说明线程并不是在执行复杂的业务计算，而是在 SQLite 客户端等待数据库操作继续。大量线程如果都停在相同的数据库调用上，线程池饥饿只是表象，真正需要查的是数据库为什么迟迟不能完成。

## SQLite_BUSY 与 WAL 合并

进一步查看代码，发现数据库连接在每次操作后都会被关闭。SQLite 使用 WAL（Write-Ahead Logging）时，WAL 文件达到一定页数，或者最后一个连接关闭时，可能触发 checkpoint，把 WAL 中的内容合并回主数据库文件。

合并过程中数据库可能暂时被锁定，其他操作就会收到：

```text
SQLITE_BUSY
```

如果大量线程同时等待这个状态，线程池会出现连锁反应：

```text
频繁关闭连接
    -> 更频繁触发 WAL 合并
    -> 数据库锁定时间变长
    -> 更多线程等待 SQLITE_BUSY
    -> 线程池线程耗尽，工作项排队
    -> 请求延迟和超时增加
```

因此，排查到 `sqlite3_prepare_v2` 并不等于 SQLite 本身“执行很慢”，还需要确认它是否正在等待锁，以及 WAL、checkpoint 和连接生命周期的状态。

## 先修复连接生命周期，再谈扩容

这类代码至少有两个问题：

1. 每次都关闭所有连接，导致 WAL 更频繁地合并，增加数据库被锁定的时间。
2. 连接池无法发挥作用，每次创建连接都要付出额外成本。

应该先复用合理的连接池，避免无必要地关闭全部连接，并根据业务特点配置 WAL、busy timeout、checkpoint 策略和事务范围。具体参数需要结合读写比例、磁盘性能与并发量验证，不能直接套用一个固定值。

更重要的是，修复连接关闭问题并不意味着线程池饥饿一定消失。SQLite 的写锁竞争在并发写入下仍然可能发生。此时把线程池调大，或者增加 CPU 线程数，通常只会让更多线程同时竞争数据库锁，反而恶化吞吐。

真正有效的方向是减少阻塞和竞争，并让每次 WAL 合并更快完成，例如：

- 缩短事务和锁的持有时间；
- 减少不必要的并发写入；
- 检查查询和索引，降低单次操作耗时；
- 提升磁盘 I/O 性能；
- 在应用层正确处理 busy、超时和重试；
- 消除 `.Result`、`.Wait()` 等同步阻塞异步代码。

## 一条可复用的排查路径

遇到“应用很慢但 CPU 不高”的问题，可以按下面的顺序收集证据：

1. 用 `dotnet-counters` 看线程池队列和线程数的趋势。
2. 用 `dotnet-stack` 找出大量线程共同停留的调用栈。
3. 如果栈集中在数据库，继续确认是查询慢、锁等待还是 I/O 慢。
4. 检查连接、事务和 WAL 生命周期，避免把数据库阻塞放大成线程池饥饿。
5. 最后再考虑线程池参数和机器资源，而不是先盲目加线程。

也可以在测试环境中使用 [Ben.BlockingDetector](https://github.com/benaadams/Ben.BlockingDetector) 帮助发现线程池中的阻塞调用。它不能替代现场指标和调用栈，但适合在问题进入生产前暴露 `.Result`、同步 I/O 等风险。
