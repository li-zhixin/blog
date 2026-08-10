---
title: Java 启动性能优化：从 Nested Jar 切换到 Exploded 部署
date: 2026-08-04
draft: false
tags:
  - Java
  - 启动性能
  - Nested Jar
  - Exploded
categories:
  - 性能优化
description: 通过将 Java 应用从单文件 Nested Jar 调整为 Exploded 目录形式，测试环境中的启动耗时降低约 36%～49%。
---

## 为什么调整打包形式

Nested Jar 会把应用自身的类和依赖 Jar 一起封装到一个可执行 Jar 中。它的优点是交付物集中、分发方便，但运行时需要通过额外的加载逻辑访问内嵌依赖。

Exploded 形式则会提前展开应用和依赖，Java 进程启动时可以直接从文件系统读取对应的类和资源，减少访问嵌套压缩包带来的开销。

## Exploded 之后如何保证 Classpath 顺序

把 Jar 展开后，一种启动方式是通过 `-cp` 指定应用类和依赖，然后直接运行应用的 `main` 方法：

```shell
java -cp "BOOT-INF/classes:BOOT-INF/lib/*" com.example.MyApplication
```

这种方式可能带来进一步的启动性能提升，但 `BOOT-INF/lib/*` 的展开顺序并不适合作为稳定的 classpath 顺序保证。一旦依赖中存在同名类或同名资源，顺序变化就可能导致运行行为与原来的可执行 Jar 不一致。

因此，这次调整仍然通过 Spring Boot 的 `JarLauncher` 启动：

```shell
java org.springframework.boot.loader.launch.JarLauncher
```

构建产物中的 `classpath.idx` 记录了依赖顺序，`JarLauncher` 会读取该文件并据此构造 classpath。这样既能获得解包后的启动性能收益，也能保持 classpath 顺序可预测，降低打包形式变化带来的兼容性风险。

Spring Boot 官方文档也有相关说明：[Unpacking the Executable JAR](https://docs.spring.io/spring-boot/3.5/reference/packaging/efficient.html#packaging.efficient.unpacking)。类似的启动性能差异也有其他用户反馈：[spring-projects/spring-boot#40125](https://github.com/spring-projects/spring-boot/issues/40125)。

## 测试结果

在同一台测试服务器上，分别使用 Nested Jar 和 Exploded 形式对三个包各测试 3 次，结果如下：

### 测试环境

| 项目 | 配置 |
| --- | --- |
| CPU 架构 | x86_64 |
| CPU 型号 | Intel Xeon Platinum 8173M @ 2.00 GHz |
| CPU 拓扑 | 112 个逻辑 CPU，2 路，每路 56 核，每核 1 线程 |
| 虚拟化 | QEMU / KVM，全虚拟化 |
| NUMA | 2 个节点，CPU 0-55 / 56-111 |
| Spring Boot | 3.5.14 |
| Java | 17 |

### 性能数据

| 包 | Nested 3 次（ms） | Nested 平均（ms） | Exploded 3 次（ms） | Exploded 平均（ms） | 改善 |
| --- | ---: | ---: | ---: | ---: | ---: |
| DESIGNER | 8344 / 7886 / 8386 | 8205.3 | 5256 / 5273 / 5264 | 5264.3 | 2941.0 ms，约 35.84% |
| SERVER_LOCAL | 9443 / 9439 / 9540 | 9474.0 | 5787 / 5703 / 5276 | 5588.7 | 3885.3 ms，约 41.01% |
| RUNTIME | 14493 / 14566 / 14571 | 14543.3 | 7271 / 7730 / 7268 | 7423.0 | 7120.3 ms，约 48.96% |

从测试结果看，三个包的启动耗时都有明显下降：

- DESIGNER 从约 8.2 秒降到 5.3 秒；
- SERVER_LOCAL 从约 9.5 秒降到 5.6 秒；
- RUNTIME 从约 14.5 秒降到 7.4 秒。

整体改善幅度在 **35.84%～48.96%** 之间。其中 RUNTIME 的绝对耗时减少约 7.1 秒，改善最明显。
