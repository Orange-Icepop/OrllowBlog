---
title: Ubuntu桌面因不支持EAP_IDENTITY而无法连接IKEv2 VPN的解决方法
categories:
  - 网络
tags:
  - Linux
  - Ubuntu
  - VPN
abbrlink: 53252
date: 2025-09-30 06:10:37
---

省流：把strongSwan装全。

## 前情提要

在Gnome桌面下，默认的图形配置界面里只支持OpenVPN/PPTP/WireGuard VPN，而无法配置基于IPsec/IKEv2的VPN（说来也奇怪，Windows下反而是内置了IKEv2和PPTP/L2TP等的客户端，而OpenVPN之类的才要额外客户端）。为了支持IKEv2，我们需要使用的是`strongSwan`，最重要的是其图形配置前端`network-manager-strongswan`。

```bash
sudo apt install network-manager-strongswan
```

但是装完之后，在证书等配置都完全正常的情况下，使用EAP-MSCHAPv2认证时会失败，报错如下：

```log
14[IKE] server requested EAP_IDENTITY (id 0x00), sending 'username'
14[IKE] EAP_IDENTITY not supported, sending EAP_NAK
15[IKE] received EAP_FAILURE, EAP authentication failed
```

{% folding cyan::tips：桌面环境下查看NetworkManager日志 %}

在桌面环境下最烦人的一点就是看不到配置出问题了之后的报错，想看也很简单：

```bash
sudo journalctl -u NetworkManager.service -f
```

tips：桌面环境下查看NetworkManager日志

在桌面环境下最烦人的一点就是看不到配置出问题了之后的报错，想看也很简单：

```bash
sudo journalctl -u NetworkManager.service -f
```

这里看到的是整个NetworkManager（Ubuntu默认的网络管理器）的日志，包括VPN配置，同时也负责所有无线/有线网络的配置，所以查一个问题的时候别经常去动网线啥的。

（说NetworkManager是默认网络管理器其实也不严谨，Ubuntu实际上的默认网络管理器是netplan，用作前端（沟槽的万能中间层还在发力），而它的后端，也就是实际管接口的东西默认是NetworkManager，如果你愿意的话也可以换成networkd，但是和Gnome桌面的适性就难说了）

{% endfolding %}

## 解决方法

还得感谢一下[这篇文章](https://wiki.strongswan.org/issues/2699)。开源社区的issue是个好地方，疑难杂症基本都能找到，就是检索的时候有时候比较麻烦（GitHub你说是吧hh）。

简单来讲，就是不光要装strongswan在NetworkManager上的前端，还得把整个strongSwan和相关附属都装全。对于服务端来讲这是常识，我也不清楚为啥Gnome这帮家伙能想出一个前端能执行大部分业务这个点子的。

```bash
sudo apt install strongswan strongswan-pki libstrongswan-extra-plugins
```

基本上装这些就全了，然后只要重启一下就正常了。

## 另一个坑点

Gnome的前端默认不会在配置里要求服务端分配IP地址，这会导致服务端不接受流量选择器（`TS_UNACCEPT`）。

解决方案是在VPN设置的“身份”选项卡里拉到底，在`Options`里勾上`Request an inner IP address`，保存并退出，就好了。