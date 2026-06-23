---
title: C#编程守则
categories:
  - 编程
tags:
  - C#
  - .NET
abbrlink: 9ec3c2b2
date: 2025-11-15 03:45:50
---

## 介绍

本页是我在学习C#时总结的一些相关编程约定，不定期扩充。

不一定要完全遵守约定，对于某些确有其事的需求，小小的不遵守一下也无伤大雅。

### 命名约定

#### 总则

详见[C#官网](https://learn.microsoft.com/zh-cn/dotnet/csharp/fundamentals/coding-style/identifier-names)，特别注意关于静态字段的命名约定。这些约定检查已经内置于JetBrains Rider等IDE中。

如果某些你认为应该使用Pascal法的字段实际上使用驼峰法命名，考虑一下是否可以将它们转换为访问器而非字段。

#### 细则

- 不要将To替换为2
- 最好不要缩写以避免全大写命名。对于实在太长以至于必须缩写的，可以设置项目级忽略
- 例外：ID建议命名为Id

### 日志相关（Microsoft.Extensions.Logging.LoggerExtensions）

- 总是使用模板而不是字符串拼接以优化性能
- 命名所有模板空位以增加可读性
- 不要在Message中使用Exception.ToString()，而是使用LogError(Exception? exception, string? string)重载