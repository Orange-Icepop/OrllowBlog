---
title: ATK设备在Linux上无法使用网页驱动的解决方案
categories:
  - 系统
tags:
  - Linux
  - ArchLinux
  - Chrome
  - udev
abbrlink: 62847
date: 2026-01-27 07:01:38
---

## 前言

最近因为有移动办公的需求，买了个ATK的无线机械键盘（感叹机械键盘市场真卷，200不到的键盘做的比我一年半之前花将近2000买的好）。选ATK一个很重要的点就是他们有支持Linux的[网页面板驱动](https://v3-hub.atkgear.com/)。

到手之后有线连接，打开浏览器，提示只支持Chrimium内核的浏览器。正常，Chromium提供的开发接口确实比Firefox多。

切到Chromium，点击`新增设备`，左上角弹出的窗口中选中ATK键盘，确定。随后我就傻眼了：在屏幕中间显示的键盘是正确的，但是状态是“休眠”，无法进行设置。

## 原因

网上搜索了一通，发现~~我想要说的前人们都说过了~~有人已经踩过坑了：

{% link Linux 下配置udev来实现chrome对于蜻蜓R1鼠标的访问::https://www.cnblogs.com/dingnosakura/p/18274600 %}

按照里面的`修改USB/HID设备权限`部分临时测试了一下，发现确实是设备权限的问题，修改为可写之后就能正常使用了，于是准备用里面提供的方法来将配置持久化。然而，进行到`添加当前用户到plugdev组`这一步时，我发现我当前的ArchLinux并没有`plugdev`这个组。抱着怀疑的心态查了一下ArchWiki，果然发现有问题：

{% blockquote ArchWiki https://wiki.archlinux.org.cn/title/Udev#Allowing_regular_users_to_use_devices %}
`OWNER`、`GROUP` 和 `MODE` udev 值可用于提供访问权限，但会遇到如何使设备对所有用户可用而不使用过于宽松的模式的问题。Ubuntu 的方法是创建一个 `plugdev` 组，并将设备添加到该组中，但这种做法不仅[不被 systemd 开发者推荐](https://bugzilla.redhat.com/show_bug.cgi?id=815093)，而且在 Arch 的 udev 规则中被视为错误（[FS#35602](https://bugs.archlinux.org/task/35602)），并且[从 systemd 258 开始已损坏](https://bbs.archlinux.org/viewtopic.php?pid=2262950)。
{% endblockquote %}

（这次我查的ArchWiki是`archlinux.org.cn`，因为`archlinuxcn.org`的这一部分没有汉化。）

原帖作者用的是基于Arch衍生的Manjaro，采取的却是这种不被建议的做法，因此我认为有必要予以纠正。

## 适用于Arch系Linux的方法

{% blockquote ArchWiki https://wiki.archlinux.org.cn/title/Udev#Allowing_regular_users_to_use_devices %}
对于 systemd 系统，现代推荐的方法是使用 `MODE 660` 来允许组使用设备，然后附加一个名为 `uaccess` 的 [TAG](https://github.com/systemd/systemd/blob/main/rules.d/70-uaccess.rules.in)。这个特殊的标签会使 udev 将一个[动态用户ACL](https://github.com/systemd/systemd/blob/main/src/udev/udev-builtin-uaccess.c)应用到设备节点上，该节点与 [systemd-logind(8)](https://man.archlinux.org/man/systemd-logind.8) 协调，以便让已登录用户可以使用该设备。有关实现此目的的 udev 规则示例，请参见：

```ini /etc/udev/rules.d/71-device-name.rules
ACTION!="remove", SUBSYSTEMS=="usb", ATTRS{idVendor}=="vendor_id", ATTRS{idProduct}=="product_id", MODE="0660", TAG+="uaccess"
```

{% noteblock warning::注意 %}
为了使任何添加 `uaccess` 标签的规则生效，其定义所在文件的名称必须在字典顺序上先于 `/usr/lib/udev/rules.d/73-seat-late.rules`。简单而言，`.rules`文件开头的数字必须是小于73的两位数（建议为71或70）。
{% endnoteblock %}

{% endblockquote %}

Wiki提供的配置和我们的目标大差不差，只需要将里面的`vendor_id`和`product_id`（均为字符串）改为ATK设备对应的即可。这两个参数可以通过`lsusb`命令来获取。

```bash
$ lsusb
# 命令输出中应当有你的ATK USB设备，格式大致如下：
Bus 004 Device 011: ID 576f:112e HFD ATK A87Pro
```

`ID`后面以冒号分隔的两段十六进制字符串，前者是`vendor_id`，后者是`product_id`。因此，最终的配置文件大致如下。记得把`vendor_id`和`product_id`都替换成你实际设备的对应信息，文件名也是。

```ini /etc/udev/rules.d/71-atk-keyboard.rules
ACTION!="remove", KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="vendor_id", ATTRS{idProduct}=="product_id", MODE="0660", TAG+="uaccess"
```

在最终的配置文件中，我们添加了`KERNEL`和`SUBSYSTEM`两个额外匹配条件，这样可以避免匹配到别的什么设备上。

至此，就可以在开机后直接使用ATK网页驱动了。

## Wine客户端？

有人可能会尝试使用wine来运行ATK HUB客户端，直接连接到键盘，但是我实际测试之后发现不行。Windows和Linux的USB设备API是截然不同的，Wine没有向应用程序暴露HID接口，更没有尝试模拟Windows的USB设备API，而这正是ATK HUB识别和操作设备的基础。因此，ATK HUB会直接找不到任何ATK设备。