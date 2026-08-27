---
title: Path.GetFullPath 与波浪号引发的路径大小写问题
date: 2026-08-13
draft: false
tags:
  - .NET
  - Windows
  - 路径处理
categories:
  - 工程实践
description: 程序使用 Path.GetFullPath 和字符串比较判断目录时，波浪号可能使路径大小写发生变化并导致误判。
---

有些程序会使用 `Path.GetFullPath()` 规范化路径，再通过 `==` 或 `StartsWith()` 判断两个目录是否相同或是否存在包含关系：

```csharp
var expected = Path.GetFullPath(Path.Combine(root, name));
var matched = expected.StartsWith(existingFolder);
```

大部分情况下它都能正常工作，`Path.GetFullPath()` 通常会保留传入路径的大小写，后续字符串比较也能通过。我们的代码跑了很多年，似乎没什么问题。

直到遇到了`~`。

假设磁盘上已有目录：

```text
D:\data
```

程序使用另一种大小写构造根目录，并枚举磁盘中已经存在的目录：

```csharp
using System;
using System.IO;

var root = @"D:\Data";
var name = "demo~cache";
var expected = Path.GetFullPath(Path.Combine(root, name));
var existing = Directory.GetDirectories(root, name)[0];

Console.WriteLine($"expected: {expected}");
Console.WriteLine($"existing: {existing}");
Console.WriteLine($"Equal: {expected == existing}");
Console.WriteLine($"StartsWith: {expected.StartsWith(existing)}");
```

在父目录实际以 `data` 保存时，可能得到：

```text
expected: D:\data\demo~cache
existing: D:\Data\demo~cache
Equal: False
StartsWith: False
```

两个字符串指向同一个 Windows 目录，但 `Path.GetFullPath()` 的结果使用了磁盘保存的 `data`，另一个路径仍使用程序传入的 `Data`。默认的 `==` 和 `StartsWith()` 区分大小写，因此判断失败。

原因是 .NET 8 在规范化路径时，只要结果包含 `~`，就会尝试调用 Windows `GetLongPathNameW` 展开短路径，并使用磁盘中的真实目录拼写。

.NET 8 相关实现可参考 [`PathHelper.Windows.cs`](https://github.com/dotnet/runtime/blob/v8.0.0/src/libraries/System.Private.CoreLib/src/System/IO/PathHelper.Windows.cs#L25-L36)。
