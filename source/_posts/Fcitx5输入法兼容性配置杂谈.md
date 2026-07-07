---
title: Fcitx5输入法兼容性配置杂谈
categories:
  - 软件
tags:
  - Linux
  - 输入法
  - ArchLinux
  - KDE
  - Wayland
abbrlink: 769a413c
date: 2025-10-25 11:53:25
---

本文包括：Arch Linux上KDE Wayland环境下Fcitx5输入法的兼容性配置，包括但不限于：Flatpak应用，Qt应用（如wemeet），Electron应用（如新QQ）。

## Wayland基础配置（共通）

{% note info %}
本段更新于2026/4/21，此时ArchWiki已更新，主要修改了关于GTK_IM_MODULE与QT_IM_MODULE的设置问题。
{% endnote %}

我相信没人还在用X11了吧？（笑）

Wayland下的fcitx5是基本开箱即用的，本段的配置是用于正常使用XWayland应用程序的。

为了支持GTK：

``` ini ~/.config/gtk-3.0/settings.ini
[Settings]
gtk-im-module = fcitx
```

{% note warning %}
请勿设置`GTK_IM_MODULE`环境变量。
{% endnote %}

为了支持Qt，设置以下环境变量：

{% note warning %}
Qt5与非KDE桌面也需要这些环境变量。
{% endnote %}

``` shell /etc/environment
QT_IM_MODULES=wayland;fcitx
QT_IM_MODULE=fcitx
```

为了支持其他的XWayland应用程序：

``` shell /etc/environment
XMODIFIERS=@im=fcitx
```

## Flatpak

{% blockquote ArchWiki https://wiki.archlinuxcn.org/wiki/Fcitx_5#flatpak %}
flatpak 沙箱应用启动时不会读到 `~/.config/gtk-3.0/settings.ini`，而是 `~/.var/app/$APP_ID/config`（受 XDG_CONFIG_HOME 控制）。

你可以使用 Flatseal 为所有 flatpak 应用（global）设定 GTK_IM_MODULE=fcitx 环境变量，也可以添加允许读取 xdg-config/gtk-3.0:ro，这样就能读到主机配置文件了。 
{% endblockquote %}

个人推荐`添加允许读取 xdg-config/gtk-3.0:ro`，因为前面那个在我这儿没用......

{% note warning %}
千万不要安装Flatpak里的`Fcitx5`包，那是一个完整的输入法，并且有一定概率顶替掉系统级软件包从而被KDE的GUI面板使用，虽然表面上看着没什么问题但是实际上会导致一切诸如主题的高级功能无法使用（因为配置文件都不知道放哪儿了）。
{% endnote %}

## Qt应用

某些Qt应用（例如腾讯会议）在未配置的情况下无法使用Fcitx5输入法。这是因为text-input-v2协议还没有整合到Wayland协议上游（Wayland这帮家伙的工作效率有些感人），在非KDE环境下或者某些没有支持这条协议的软件还是需要配置`QT_IM_MODULE=fcitx`来采用传统方法让软件使用输入法。KDE中，直接在开始菜单里右键应用图标，选择“编辑应用程序”，在“环境变量”一栏中填入即可。

{% asset_img wemeet-ime.webp 腾讯会议 %}

[Fcitx官方文档](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland#Qt5_.2F_Qt6)有时候还是非常有用的。

PS:前面的环境变量`__EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/50_mesa.json`也是腾讯会议相对重要的一个配置，否则在某些双显卡的笔记本下，使用独显直连模式时会导致无法观看他人的屏幕共享。混合模式就随便了。

## Electron应用

Electron这玩意儿属实臭名昭著，但是谁叫它用的是几乎无缝的JavaScript技术栈呢？QQ就是要用Electron，腾子才不会管你对它怨言多大。

同样根据[官方文档](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland#Chromium_.2F_Electron)，只要在运行参数中添加`--enable-features=UseOzonePlatform --ozone-platform=wayland --enable-wayland-ime`即可。

{% asset_img qq-ime.webp 新版QQ %}

需要注意的是，某些教程会让你使用包含`--ozone-platform-hint=auto`的启动参数，在我这里无法使用，因为这样的话QQ会跑在X11模式下而非原生Wayland，现象就是除非在环境变量里设置`XMODIFIERS=@im=fcitx`，也就是使用XWayland，否则还是无法使用输入法。

## 其它

### GTK应用

默认情况下使用没碰到问题，暂时不讲。

### JetBrains IDE

JetBrains一系的IDE都是用Java写的，正是这一点造就了它强大的跨平台能力。

しかし，Wayland的支持可以说是有进度但是非常缓慢，特别是输入法这块，至少早在2023年就有人提交了issue请他们支持原生Wayland输入法，然而这两年过去了，15天前（2025/10/10）才有确切消息说2025.3版本会部分支持Wayland输入法，“对周围文本API的支持”（别问我啥意思我也不知道，但是应该是挺重要一个功能）则计划推迟到2026.1版本。

2025.3这个版本已经进公开测试版了，急的可以先从AUR下一个JetBrains Toolbox装一装测试版，其他人就等一手正式版更新吧，AUR应该会立马跟上，我也会到时候更新。

在XWAYLAND模式下，把`XMODIFIERS=@im=fcitx`加进应用程序的环境变量里就好了。

---

#### 2025/11/14更新

JetBrains Rider 2025.3版本已经能够在原生Wayland模式下使用中文fcitx5输入法。

帮助-编辑自定义虚拟机选项，在尾部添加一行：

```ini
-Dawt.toolkit.name=WLToolkit
```

然后重启IDE即可。