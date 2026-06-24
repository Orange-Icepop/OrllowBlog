---
title: iStoreOS防火墙放行特定IPv6流量进入内网
categories:
  - 网络
tags:
  - iStoreOS
  - 防火墙
  - IPv6
  - 软路由
abbrlink: 6767a142
date: 2025-08-04 12:44:31
---

因为爱快用的太糟心，我将家里的主网关换成了iStoreOS的系统，但是两个系统的操作逻辑不大一样，导致很多配置的迁移遇到了比较大的阻力。几个例子：

- IPv6解析与通行默认阻断，设置藏得很深
- 自带的DDNS很容易出问题
- 内网DHCP设置不在网络-DHCP/DNS而是在网络-接口
- ......

这里要详细展开的是关于iStoreOS（其实带LuCI的OpenWRT都一样）的防火墙如何有限地放行IPv6流量。

iStoreOS版本：24.10.2-2025070410

## 为什么要放行IPv6流量？

默认情况下，对于外网主动访问的普通流量，OpenWRT的策略是直接阻断，这样能够保证内网中具有公网IP的设备（下文特指IPv6，毕竟国内没几个人家里内网能全是公网IPv4的吧……）不会在你没允许的情况下被外网访问。

然而，如果你内网有需要对外开放的应用，比如NAS或者博客，那么放行IPv6就相当的有必要，因为许多网关完全不支持IPv6栈的端口转发（不过iStoreOS倒是支持），毕竟公网v6唾手可得。更重要的是，这是一个学习OpenWRT防火墙策略与配置的大好机会。

## 位置

侧栏：网络-防火墙

{% asset_img istoreos-firewall.webp 防火墙 %}

## 关键配置

我们要首先知道的一点是，OpenWRT的端口转发与通信规则优先级是高于区域设置的，也就是说在“仅开放部分端口”的需求下是不需要配置区域设置这一基本配置的，毕竟我们的默认要求就是阻断一切不允许的流量。

理解了这一点之后，我们的目标就很明确了。进入顶部的“通信规则”板块，拉到底部，点击添加，进行如下配置：

{% asset_img ipv6-allow.webp 允许IPv6通过 %}

NAS和网站的协议基本上都是TCP，因此实际上UDP不是必须的。

源区域选择整个WAN区域即可，目标区域则是LAN区域。

目标端口按照个人需求填写，使用英文减号连接端口区间的两个端点，用英文逗号分隔单个端口或端口区间。

这里需要特别注意的是目标地址。

## IPv6地址后缀匹配

使用公网IPv6最让人头疼的一点莫过于动态IP了。和动态公网IPv4不同，内网设备的动态公网IPv6地址在网关处根本无法获取。为了解决这一点，我们必须启用EUI-64这一静态后缀分配策略。

{% folding cyan::吐槽 %}

作者写到这里都开始怀疑自己做的是否太复杂了……NAT6一开配个静态IP开个端口转发直接万事无虞。但是这样子一方面网关还要处理v6栈的NAT，对于一颗J1900来讲还是一定的负担，又会增加延迟，还要在网关配置一大把端口转发……还是算了。

{% endfolding %}

EUI-64简单来讲就是一个国际标准规则，使用MAC地址生成IPv6地址（二进制形式）的末64位，这样只要不更换网口，IPv6地址的末64位就永远是固定的，让网关能够通过后缀匹配来识别出到服务器的流量与其它非法访问流量。

我们假设MAC地址为 `7F:1A:2B:3C:4D:5E`（十六进制）。

第一步，在正中间插入`FF:FE`，即`7F:1A:2B:FF:FE:3C:4D:5E`。

第二步，翻转第一个字节的第七位（称为`U/L`位），也就是最开始的两位十六进制数。本例中的第一个字节是`7F`，其二进制形式为`01111111`。翻转第七位后是`7D`（`01111101`）。

最终的接口ID即为`7D:1A:2B:FF:FE:3C:4D:5E`。

在OpenWRT中，使用双冒号开头+掩码来表示后缀匹配，即`::7d1a:2bff:fe3c:4d5e/::ffff:ffff:ffff:ffff`。

{% note warning %}
## 关于OpenWRT上后缀匹配的语法

Linux防火墙中进行IPv6后缀匹配的标准语法就是`::后缀/::掩码`，如果不这么写，只写`::后缀`，会导致没有流量匹配该规则，设了等于白设。
参考该[github issue](https://github.com/istoreos/istoreos/issues/2642)

{% endnote %}

除了EUI-64，还有一种方案是直接配置静态后缀，最终效果和EUI-64是一样的，总体上更接近IPv4的内网配置，但是单一性保证相对就没有那么高。

## Windows开启EUI-64

Windows自从很久之前开始就开始使用随机临时IPv6地址机制来保护系统安全，作为客户端的时候几乎是无感的免费安全性保证，而这种时候反而成了累赘。

在管理员权限的Powershell中，执行`Get-NetIPv6Protocol`命令，输出应该类似如下：

```bash
DefaultHopLimit : 128
NeighborCacheLimit(Entries) : 256
RouteCacheLimit(Entries) : 4096
ReassemblyLimit(Bytes) : 266178496
IcmpRedirects : Enabled
SourceRoutingBehavior : DontForward
DhcpMediaSense : Enabled
MediaSenseEventLog : Disabled
MldLevel : All
MldVersion : Version2
MulticastForwarding : Disabled
GroupForwardedFragments : Disabled
RandomizeIdentifiers : Enabled
AddressMaskReply : Disabled
UseTemporaryAddresses : Enabled
MaxTemporaryDadAttempts : 3
MaxTemporaryValidLifetime : 7.00:00:00
MaxTemporaryPreferredLifetime : 1.00:00:00
TemporaryRegenerateTime : 00:00:05
MaxTemporaryDesyncTime : 00:10:00
DeadGatewayDetection : Enabled
```
主要关注`RandomizeIdentifiers`和`UseTemporaryAddresses`两个参数，将它们都改为Disabled：

```powershell
Set-NetIPv6Protocol -UseTemporaryAddresses Disabled
Set-NetIPv6Protocol -RandomizeIdentifiers Disabled
```

然后重启，或者禁用再启用对应的网络适配器，EUI-64规则就生效了。

## Linux开启EUI-64

各个Linux发行版使用的网络接口管理器各不相同，很难在一篇文章里一次性讲清楚。这里只讲解新版Ubuntu使用的netplan前端+NetworkManager后端的实现方案。

netplan的配置文件在`/etc/netplan/`目录下的yaml文件中。以下是我的配置示例：

```yaml
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

其中enp2s0代表的是以太网接口名称。在它的下一级（与match键同级）添加`ipv6-address-generation: "eui64"`：

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp2s0:
      ipv6-address-generation: "eui64"
      match:
        macaddress: "AA:BB:CC:DD:EE:FF"
      addresses:
      - "192.168.2.7/24"
      routes:
      - to: "0.0.0.0/0"
        via: "192.168.2.1"
```

然后输入`sudo netplan try`检查有没有问题。没有问题的话，会提示是否应用，按enter即可应用。

## 默认阻断

写完这篇文章前面部分不久之后，我发现网关配置里的默认区域转发配置不知道为啥被动过了，`DENY all others`变成了`ACCEPT all others`，导致有人进了我的ssh关掉了MC服务器（谁这么恶趣味啊），然后找了半天没找到怎么调回去……最后发现只要把默认转发配置改成拒绝就好了：

{% asset_img istoreos-default-deny.webp 默认转发拒绝 %}