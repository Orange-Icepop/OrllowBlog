---
title: sched-ext在不同电源模式下的切换方法
categories:
  - 系统
tags:
  - Linux
  - systemd
  - ArchLinux
abbrlink: 41d012f5
date: 2026-06-24 22:32:25
---

## sched-ext简介

自 Linux 6.12 起，内核开始支持名为 `sched-ext` 的协议，它的作用是，允许用户使用自定义的CPU调度器覆盖内核原有的调度器，以实现更具有场景针对性的调度策略。与此同时，这个调度器可以在运行时快速热切换，而相比之下原本的调度器写死在内核中，想要更改只能重编译内核。此功能目前主要由 `scx-scheds` 程序实现。

## 我的选择

我在我的 ArchLinux 笔记本上使用的是 `scx_bpfland` 调度器，这个调度器对12代的大小核有比较好的优化：

{% blockquote Deepseek %}

- 为什么适合大小核：bpfland 基于自愿上下文切换来精准识别“交互式任务”（如鼠标点击、窗口拖动、打字）。它会自动将这些高优先级交互任务提升至高性能大核（P核），同时将后台编译、日志同步等非交互任务挤压到能效小核（E核）上运行。

- 功耗表现：它的调度算法非常“克制”，不会盲目拉高CPU频率。在待机或轻载办公时，任务会尽可能留在E核，P核保持深度休眠，对笔记本续航非常友好。

- 交互流畅度：公认的“丝滑感”调度器，桌面响应速度极佳，在 sched_ext 社区是笔记本用户的口碑首选。

- 状态：生产就绪，稳定可靠。

{% endblockquote %}

如何安装scx-scheds可以参考[ArchWiki](https://wiki.archlinuxcn.org/wiki/Scx-scheds)，并请配置默认调度器（推荐bpfland，当然你也可以按需修改）。切记使用 `scx_loader.service` 而不是 `scx.service` 。

{% note info %}

在RHEL系系统上，安装scx-scheds需要启用额外仓库。

Fedora用户可以使用以下命令：

```shell
sudo dnf copr enable bieszczaders/kernel-cachyos-addons
sudo dnf install scx-scheds
```

EL 10 架构的发行版（如Rocky Linux 10）可以启用一个专门为 EL 10 编译了相关包的仓库：

```shell
sudo dnf copr enable andersrh/kernel-cachyos-addons-el10
sudo dnf install scx-scheds
```

COPR 为社区仓库，启用请自行负责。

由于 EL 10 使用的刚好是 Linux 6.12，因此更老的 EL 发行版无法使用sched-ext，也就不用想着安装了。

{% endnote %}

## 模式切换

几乎所有sched-ext调度器都提供了多种调度模式（共有的模式有 `Auto` `LowLatency` `PowerSave` `Gaming` ，`bpfland` 独有一个 `Server` 模式），在笔记本电脑上，不同的工作环境往往需要不同的调度策略。`scx-scheds` 提供了一个CLI工具 `scxctl` 能够对目前正在运行的调度器进行修改，包括热切换调度器、切换调度器工作模式等。

{% note info %}
`scxctl` 通过D-Bus接口 `org.scx.Loader` 与托管在systemd上的scx_loader.service进行交互。该服务默认情况下被配置为按需启动，换句话说如果开机时没有调用`scxctl`可能导致调度器不生效。因此，建议将 `scx-loader.service` 设置为开机自启。
{% endnote %}

`scxctl` 基础用法如下：

```shell
# 列出所有当前支持的sched-ext调度器
scxctl list
# 切换调度器（需要sudo/root，此处示例为切换到bpfland）
scxctl switch --sched bpfland
# 更改工作模式（需要sudo/root）
scxctl switch --mode auto
# 也可以同时操作
scxctl switch --sched bpfland --mode gaming
```

## 根据电源模式自动切换调度器工作模式

我使用的笔记本为联想拯救者，有一个很方便的功能是可以通过 `Fn + Q` 来快速切换电源模式。在Linux下，这个功能由`power-profiles-daemon`承担，可以通过监听它的D-Bus消息来获知系统电源模式修改事件，并通过解析事件内容来对应地修改调度器的工作模式。完整的Python代码如下：

```python /usr/local/bin/ppd-watcher.py
#!/usr/bin/env python3
import gi
gi.require_version('GLib', '2.0')
gi.require_version('Gio', '2.0')
from gi.repository import GLib, Gio
import subprocess

MODE_MAP = {
    'performance': 'gaming',
    'balanced':    'auto',
    'power-saver': 'powersave'
}

def on_properties_changed(connection, sender_name, object_path, interface_name, signal_name, parameters, user_data):
    # 只处理 PropertiesChanged
    if signal_name != 'PropertiesChanged':
        return
    # 参数结构: (interface_name, changed_properties, invalidated_properties)
    # parameters 是 GLib.Variant 元组，第一个元素是接口名称（即 net.hadess.PowerProfiles）
    if parameters.n_children() < 3:
        return
    # 获取第二个元素，即 changed_properties 字典
    changed = parameters.get_child_value(1).unpack()
    if 'ActiveProfile' in changed:
        new_profile = changed['ActiveProfile']
        print(f"Power profile changed to: {new_profile}")
        mode = MODE_MAP.get(new_profile)
        if mode is None:
            print(f"Unknown profile: {new_profile}, ignoring")
            return
        # 执行切换命令
        cmd = ['scxctl', 'switch', '--mode', mode]
        try:
            subprocess.run(cmd, check=True, capture_output=True, text=True)
            print(f"Switched bpfland to mode: {mode}")
        except subprocess.CalledProcessError as e:
            print(f"Switch failed: {e.stderr}")

def main():
    bus = Gio.bus_get_sync(Gio.BusType.SYSTEM, None)
    # 订阅 PropertiesChanged 信号，精确匹配服务名、路径和信号
    bus.signal_subscribe(
        'net.hadess.PowerProfiles',          # sender
        'org.freedesktop.DBus.Properties',   # interface
        'PropertiesChanged',                 # member
        '/net/hadess/PowerProfiles',         # path
        None,                                # arg0 (无额外过滤)
        Gio.DBusSignalFlags.NONE,
        on_properties_changed,
        None
    )
    print("Listening for power profile changes on net.hadess.PowerProfiles...")
    loop = GLib.MainLoop()
    loop.run()

if __name__ == '__main__':
    main()

```

开头的shebang可以让该脚本直接在shell中运行，无需手动指定解释器。

{% note info %}
### 为什么要用Python而不是shell脚本

有人可能顾虑Python解释器的内存/CPU开销比较大，希望转而使用shell脚本进行运行。实际上这是必要的取舍。

首先，Python的内存开销没有那么大，基本上在10MB的量级，对于一个现代系统而言尚处于可接受的范围（electron大爹创过来咯）。

其次，Python的`DBus`和`GObject`库提供了安全的DBus高级抽象，鲁棒性比较好，否则要么得用shell硬解二进制DBus消息，要么得调用外部工具（换句话说就是消耗更多内存），得不偿失。

除此之外，`GLib.MainLoop()` 实际上并不是真的跑了个循环，而是使用了 `poll()` 系统调用，没有工作的循环都被优化掉了，因此空闲时CPU实际占用为0%，完全不需要担心CPU开销。

{% endnote %}

运行该脚本需要`dbus`和`gobject`两个Python库，请使用系统包管理器安装。在ArchLinux上，命令如下：

```shell
sudo pacman -S python-dbus python-gobject
```

将该脚本以root所有权放置在 `/usr/local/bin/ppd-watcher.py` ，并标记为可执行。

然后编写systemd单元：

```ini /etc/systemd/system/ppd-watcher.service
[Unit]
Description=Watch power profile changes and switch bpfland scheduler
After=power-profiles-daemon.service
Requires=power-profiles-daemon.service

[Service]
Type=simple
Environment="PYTHONUNBUFFERED=1"
ExecStart=/usr/local/bin/ppd-watcher.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

{% note info %}
关于为什么要设置 `Environment="PYTHONUNBUFFERED=1"` 环境变量，请参考[这篇文章](/posts/be8d524e)。
{% endnote %}

配置开机自启并立即启动：

```shell
sudo systemctl daemon-reload
sudo systemctl enable --now ppd-watcher.service
```

使用 `systemctl status ppd-watcher.service` 检查启动情况，如果出了问题请检查脚本的执行权限有没有打开。