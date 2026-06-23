---
title: Python脚本无法输出stdout到journald解决方案
categories:
  - 系统
tags:
  - systemd
  - Linux
abbrlink: be8d524e
date: 2026-06-23 06:51:10
---

## 现象

当一个 systemd 服务的执行内容为一个python脚本时，在journalctl中无法观察到python脚本的stdout内容。

## 原因

这是因为 Python 默认对 stdout 使用行缓冲，但在 systemd 环境下（非交互式终端）变成了全缓冲，导致日志在进程退出前才刷新。（骗你的，甚至可能退出了也不刷新）

## 解决方案

两种解决方案：

### 1. 添加环境变量

最方便的一个方案，添加一个`PYTHONUNBUFFERED`（Python unbuffered）环境变量，可以直接解决这个问题。
systemd单元文件支持Environment参数，添加起来很方便。

``` ini
[Service]
Environment="PYTHONUNBUFFERED=1"
```

### 2. 手动刷新

在脚本中每次 print 后手动刷新。例如，添加 `sys.stdout.flush()`，或使用 `print(..., flush=True)`。

（我没试过方案2，方案1直接把事情解决了哈哈哈）

至此，问题解决。
