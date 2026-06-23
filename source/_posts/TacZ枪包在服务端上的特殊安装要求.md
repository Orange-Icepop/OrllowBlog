---
title: TacZ枪包在服务端上的特殊安装要求
categories:
  - MC
tags:
  - Mod
  - TacZ
  - 服务器
abbrlink: 39381
date: 2025-08-04 23:11:07
---

TacZ枪包在新版客户端上的安装大家都明白，直接把zip包丢到版本文件夹底下的`tacz`文件夹里面就可以了。然而，这一举措（放在服务器根目录下的`tacz`文件夹中）在某些服务端上是不可行的，会造成枪包无法加载（`/tacz reload`根本读不到）。

在放置好枪包后第一次启动服务器之后，位于服务器根目录下的`tacz`文件夹中所有枪包均会被移动到一个同样名为`tacz`的文件夹中，而原文件夹中会多出一份名为`tacz-pre.toml`的文件，其内容如下：

``` toml
[gunpack]
    #When enabled, the mod will not try to overwrite the default pack under .minecraft/tacz
    #Since 1.0.4, the overwriting will only run when you start client or a dedicated server
    DefaultPackDebug = false
```

将`DefaultPackDebug`的值改为`true`之后，枪包就不会被移动了。此时将被挪走的枪包挪回来，然后重启服务器，就可以正常加载枪包了。