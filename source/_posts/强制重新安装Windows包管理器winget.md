---
title: 强制重新安装Windows包管理器winget
date: 2025-07-06T21:37:58+08:00
categories: 
  - 软件
tags: 
  - Windows
  - Appx
---

## 省流

``` Powershell
Add-AppxPackage {Microsoft.DesktopAppInstaller的appx安装包路径} -ForceApplicationShutdown -ForceUpdateFromAnyVersion
```

## 前情提要

新买了一块1TB的系统盘，用傲梅分区助手做了一下系统迁移，进系统一看，发现Microsoft Store的应用怎么也没法安装了。

使用Add-AppxPackage呢？提示：找不到指定卷UUID。

反手一个查注册表，得，系统盘UUID被改过了。我是把系统分区和软件分区放在同一块物理盘上的，这两个分区在系统迁移之后UUID全变了。

手动改回来呢？重启完之后没有改善。按照Deepseek的指示删掉了整个记录UUID的项，重启，这下应用安装程序也不报错误码了，直接跟我说解析应用包时出错。

还能怎么办？重装系统解决99%的问题，直接保留文件重装。

重装完之后，在Microsoft Store里倒是可以正常安装了，但是应用安装程序还是照旧。

## 尝试重装时发生的问题

既然现在能装UWP应用了，说明Appx系统算是修好了，问题就单纯出在应用安装程序上。Microsoft Store大概因为自带Deploy方法，或者是直接调用Add-AppxPackage的，所以没有问题。

应用安装程序的Appx标识符是Microsoft.DesktopAppInstaller。先尝试Remove-AppxPackage，报错：此应用是Windows系统的一部分，无法卸载。

后来甚至用上了把系统内置镜像的包卸载掉，然并卵。

还能怎么办？搞不好又要把系统搞坏了……先搁置一下。

## 解决过程

搁置了几天，某天装一个需要用winget安装的程序，反手丢给我一个报错：winget.exe没有应用包标识符。emmmm……摆烂了，直接查微软官方文档怎么重新安装，跟我说去github上下载发行包。

一看，这熟悉的msixbundle，不是还是Appx包吗？

{% asset_img desktop-installer-gh.webp GitHub上的发行版列表 %}

给我气了个半死，但是还能咋办？只好还是下下来。

先尝试直接上Add-AppxPackage，报错：应用正在运行。

还好我们的牢软准备了相当完善的[操作手册](https://learn.microsoft.com/zh-cn/powershell/module/appx/add-appxpackage)，直接上去看一眼。

哦吼，发现了几个看起来很危险的参数哦~

{% asset_img desktop-installer-options.webp %}

事不宜迟，就先试试看……

欸？这就重装好了？

{% asset_img winget-v.webp %}

......6

鉴定为还是过于依赖网络安装导致的。

{% asset_img 放弃分析.webp %}