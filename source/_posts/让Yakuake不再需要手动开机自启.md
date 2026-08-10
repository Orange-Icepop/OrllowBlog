---
title: 让Yakuake不再需要手动开机自启
categories:
  - 软件
tags:
  - KDE
  - Linux
abbrlink: bc72859e
date: 2026-08-10 05:38:43
---

## 问题缘起

Yakuake 是一款下拉终端，一个快捷键就可以便捷呼出，并且由于强制的标签页系统，可以做到多开时极低的内存占用。然而它有一个经典的问题：**快捷键只有在 Yakuake 已经启动的情况下才能呼出或隐藏窗口**。换句话说，如果你刚开机还没手动启动过 Yakuake，按下快捷键什么都不会发生。如果开机后手动点一下图标启动，每次开机后还要多一个“先启动 Yakuake”的步骤，实在是别扭。

基础的解决方案是开机自启，把 Yakuake 塞进桌面环境的自动启动列表里。但我不喜欢这种做法，因为假如途中Yakuake因为各种原因退出了，重启就还得手动启动。我更希望的是：**按下快捷键时，如果 Yakuake 没启动就自动启动并展开；如果已经启动就正常切换显隐**。

## 探索过程

### 方案一：脚本 + sleep

最直接的思路是写一个脚本，先检测 Yakuake 是否在运行，没运行就启动它，然后通过 D-Bus 调用来切换窗口状态：

```bash
#!/bin/bash

if pgrep -x "yakuake" > /dev/null; then
    dbus-send --type=method_call --dest=org.kde.yakuake \
        /yakuake/window org.kde.yakuake.toggleWindowState
else
    yakuake &
    sleep 1
    dbus-send --type=method_call --dest=org.kde.yakuake \
        /yakuake/window org.kde.yakuake.toggleWindowState
fi
```

这个脚本的思路很清晰，但有一个致命缺陷：Yakuake 启动后需要一段时间才能完成 D-Bus 服务的注册。如果在启动后立即调用 `dbus-send`，会因为服务尚未就绪而失败。于是不得不在启动后加一个 `sleep 1`。

`sleep` 方案虽然能用，但本质上是在赌，在慢速机器上或者卡了的时候1秒可能还不够，得再按一遍快捷键。这显然不够优雅。

### 方案二：利用 Yakuake 自带的设置

后来翻了一下 Yakuake 的设置，发现了一个选项：**“Yakuake启动后展开窗口”**。

这个选项的作用是：Yakuake 启动后自动展开终端窗口，而不是停留在后台等待用户按快捷键。开启后，脚本逻辑可以大幅简化：

```bash
#!/bin/bash

if pgrep -x "yakuake" > /dev/null; then
    dbus-send --type=method_call --dest=org.kde.yakuake \
        /yakuake/window org.kde.yakuake.toggleWindowState
else
    yakuake &
fi
```

不再需要 `sleep`，也不再需要在启动分支里额外调用 D-Bus。启动 Yakuake 后，它自己会处理好展开的事情。脚本只需要负责两件事：检测是否已运行，以及调用 D-Bus 切换窗口状态。

写完了这个脚本之后把它放在合适的位置（我直接把它丢到`/opt`下面了），赋予执行权限，然后把Yakuake的快捷键取消掉，转而让这个快捷键启动这个脚本就行。在KDE Plasma上，所有设置都在`键盘-快捷键`下面，点击右上角的`添加`，选择`命令或脚本`，选中这个脚本，然后在右侧添加快捷键即可。

需要注意的是，默认的`F12`快捷键容易与包括浏览器的开发者面板在内的各种应用快捷键冲突，我设置的是`Win+Tab`，因为它的行为和`Alt+Tab`一样，没有存在两个的必要。

最终方案的核心思路是：**把“展开”这件事交给 Yakuake 自己处理，脚本只负责“启动”和“切换”**。善用软件自带的选项，往往比在脚本里做各种 workaround 更优雅、更可靠。
