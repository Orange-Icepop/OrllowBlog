---
title: Linux通过systemd指定网卡名称
categories:
  - 系统
tags:
  - Linux
  - systemd
abbrlink: b197b0f0
date: 2026-08-10 14:15:29
---

{% note info %}

这是[netplan方案](/posts/5d7e2e40/)在其他发行版上不适用的情况下的终极解决方案，适用于有systemd的发行版。没有systemd的发行版可以使用udev规则。

{% endnote %}

没想到网卡名称随便乱动的毛病不只在Ubuntu上有，几乎所有发行版都有，当然也有可能是我设备犯了一些什么毛病导致的。

通用的方案是使用systemd link文件，通过MAC地址来匹配网卡。原理是利用systemd作为PID1的特殊权限，在内核工作完成后、所有应用与服务启动前匹配并修改网卡名称。这是最现代、最清晰的方法。

## 第一步：找到网卡的 MAC 地址

首先，使用 `ip link` 命令列出所有网络接口，找到你想要修改名称的网卡及其 MAC 地址。

```bash
ip link show
```

输出会类似这样：

```text
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:00:5e:00:53:1a brd ff:ff:ff:ff:ff:ff
```

记录下 `link/ether` 后面的 MAC 地址（例如 `00:00:5e:00:53:1a`）。

## 第二步：创建并编辑 systemd link 文件

在 `/etc/systemd/network/` 目录下创建一个 `.link` 文件。文件名建议以 `70-` 开头，便于管理。
```bash
sudo vim /etc/systemd/network/70-custom-eth.link
```

填入以下内容，将 `00:00:5e:00:53:1a` 替换为你的 MAC 地址，将 `eth0` 替换为你想要的新名称。

```ini
[Match]
MACAddress=00:00:5e:00:53:1a

[Link]
Name=eth0
```

`[Match]` 部分用于匹配目标网卡。这里使用 `MACAddress=` 是最可靠的方式。

`[Link]` 部分的 `Name=` 用于指定新的接口名称。

## 第三步：使配置生效

完成配置后，需要重启系统或重启 `systemd-udevd` 服务来应用更改。推荐使用前者，这样更加彻底并且可以规避设备忙的问题。

### 补充说明与注意事项

虽然可以将新名称使用 `eth0` 这类名称，但官方建议避免使用，因为它们可能在启动时引发内核和 udev 的竞争条件。建议使用更具描述性的名称，如 `internal0` 或 `provider0`。我个人使用的是`wan0`。

按照以上步骤操作，就可以成功地使用 systemd 为指定 MAC 地址的网卡修改名称了。