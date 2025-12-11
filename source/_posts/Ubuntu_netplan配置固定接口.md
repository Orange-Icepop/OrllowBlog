---
title: Ubuntu netplan配置固定接口
date: 2025-07-10T21:43:12+08:00
categories: 
  - 软件
tags: 
  - Linux
  - Ubuntu
  - 网络
  - netplan
---

我不知道各位有没有碰到过类似的问题：你刚刚把你的Ubuntu计算机加好新硬件，插电开机（冷启动），然后SSH一登，Connection Timeout，回到本地一看，netplan给你写了份新的配置文件导致IP不同……

{% asset_img invasion.webp 册那！ %}

究其原因，一个比较有说服力的解释是某些玄学的网口卸载逻辑导致冷启动时系统误以为这个接口已经有网口占着了，然后就自己创建了一份新的配置。实测下来这种情况基本上只在Ubuntu Desktop上出现，因为ubuntu-desktop才有这么一套自动配置机制。

想要解决其实相当简单，溜一遍参考文档，找到了一个好东西：`match`键。通过这个玩意儿，我们可以直接将一份netplan配置绑定到指定MAC地址的设备上（当然也可以通过设备名绑定，但是那样不大符合我们的初衷）。

例如，我的netplan配置文件（位于`/etc/netplan`下面的某个yaml文件）是这样的：

``` yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp2s0:
      match:
        macaddress: "AA:BB:CC:DD:EE:FF"
      addresses:
      - "192.168.2.7/24"
      routes:
      - to: "0.0.0.0/0"
        via: "192.168.2.1"
```

其中，match键就是用来匹配设备的。只要把你的MAC地址填到`macaddress`里，冷启动时系统就会认得这个设备，然后就不会再自己创建新的配置文件了。
