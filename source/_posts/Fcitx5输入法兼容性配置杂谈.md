---
title: Fcitx5输入法兼容性配置杂谈
date: 2025-10-25T19:53:25+08:00
categories: 
  - 软件
tags: 
  - Linux
  - 输入法
  - ArchLinux
  - KDE
  - Wayland
---

本文包括：Arch Linux上KDE Wayland环境下Fcitx5输入法的兼容性配置，包括但不限于：Flatpak应用，Qt应用（如wemeet），Electron应用（如新QQ）。

## Flatpak

Flatpak本质上是提供了一层封装，让里面的所有应用都在容器里运行，因此软件不具有直接访问输入法接口的能力。需要在Flatpak中安装"Fcitx 5"包才能提供这种桥接功能，并且这个包也只有桥接的功能，宿主系统上还是需要安装完整的Fcitx5输入法才能使用。

## Qt应用

某些Qt应用（例如腾讯会议）在未配置的情况下无法使用fcitx5输入法。KDE授意在Wayland下不配置GTK_IM_MODULE和QT_IM_MODULE全局环境变量，但是有些软件依赖该配置。我们也不需要打破这条规则，直接在开始菜单里右键应用图标，选择“编辑应用程序”，在“环境变量”一栏中填入即可。

{% asset_img wemeet-ime.webp 腾讯会议 %}

需要注意的是，QT_IM_MODULE环境变量应当设置为fcitx而非fcitx5，否则将仍然无法使用输入法。[官方文档](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland#Qt5_.2F_Qt6)有时候还是非常有用的。

PS:前面的环境变量__EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/50_mesa.json也是腾讯会议相对重要的一个配置，否则在某些双显卡的笔记本下，使用独显直连模式时会导致无法观看他人的屏幕共享。

## Electron应用

Electron这玩意儿属实臭名昭著，但是谁叫它用的是几乎无缝的JavaScript技术栈呢？QQ就是要用Electron，腾子才不会管你对它怨言多大。

同样根据[官方文档](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland#Chromium_.2F_Electron)，只要在运行参数中添加--enable-features=UseOzonePlatform --ozone-platform=wayland --enable-wayland-ime即可。

{% asset_img qq-ime.webp 新版QQ %}

需要注意的是，某些教程会让你使用包含--ozone-platform-hint=auto的启动参数，在我这里无法使用，因为这样的话QQ会跑在X11模式下而非原生Wayland，现象就是除非在环境变量里设置XMODIFIERS=@im=fcitx5，也就是使用XWayland，否则还是无法使用输入法。

## 其它

### GTK应用

默认情况下使用没碰到问题，暂时不讲。

### JetBrains IDE

JetBrains一系的IDE都是用Java写的，正是这一点造就了它强大的跨平台能力。

しかし，Wayland的支持可以说是有进度但是非常缓慢，特别是输入法这块，至少早在2023年就有人提交了issue请他们支持原生Wayland输入法，然而这两年过去了，15天前（2025/10/10）才有确切消息说2025.3版本会部分支持Wayland输入法，“对周围文本API的支持”（别问我啥意思我也不知道，但是应该是挺重要一个功能）则计划推迟到2026.1版本。

2025.3这个版本已经进公开测试版了，急的可以先从AUR下一个JetBrains Toolbox装一装测试版，其他人就等一手正式版更新吧，AUR应该会立马跟上，我也会到时候更新。

在XWAYLAND模式下，把XMODIFIERS=@im=fcitx加进应用程序的环境变量里就好了。

---

#### 2025/11/14更新

JetBrains Rider 2025.3版本已经能够在原生Wayland模式下使用中文fcitx5输入法。

帮助-编辑自定义虚拟机选项，在尾部添加一行：

```ini
-Dawt.toolkit.name=WLToolkit
```

然后重启IDE即可。