---
title: Mihomo fakeip模式作为DNS服务器导致未设置代理软件无法上网问题解决
categories:
  - 网络
tags:
  - Linux
  - 代理
  - mihomo
abbrlink: 4d9d4205
date: 2026-07-15 12:51:37
---

## 前情提要

由于smartdns在ArchLinux上安装并设置为开机自启后，会较大地拖慢系统到达graphical.target的时间（在critical-chain上显示会增加1s+），再加上同有DNS服务的mihomo也本来就是开机自启的，遂让mihomo的DNS监听`127.0.0.1:5053`，并使用`systemd-resolved`将其配置为系统DNS上游。

```yaml /etc/mihomo/config.yaml
dns:
  enable: true
  prefer-h3: true
  listen: 127.0.0.1:5053
  ipv6: true
  use-hosts: true
  use-system-hosts: true
  enhanced-mode: fake-ip
```

这么配置后，浏览器访问所有网站均正常，但是本地的大部分软件都无法正常联网了。

## 工作原理

为了理解这个问题是怎么发生的，我们需要理解mihomo代理的工作原理。

以下是传统的redir-host模式在仅代理模式（不承担系统DNS工作）下的工作方式：

{% mermaid %}
sequenceDiagram
  box 本地设备与代理
  participant CP as 代理客户端
  participant M as mihomo
  end
  box 公网
  participant PDNS as 公共DNS服务器 
  participant PS as 代理服务器
  participant WEB as 互联网
  end
  CP->>M: 代理DNS请求
  M->>PDNS: DNS请求
  PDNS-->>M: DNS回复
  M-->>CP: DNS回复并记录
  critical
    CP->>M: 使用代理协议访问网络
  option 分流直连
    M->>WEB: DIRECT直连
  option 分流代理
    M->>PS: PROXY代理
    PS->>WEB: 代理访问互联网
  end
{% endmermaid %}

在这种情况下，没有配置代理的软件直接使用公共DNS或者smartdns，直连公网，没有问题。

fake-ip模式的思路则是这样的：既然配置了代理的软件的流量必然要经过mihomo，为什么一定要返回真的DNS返回值呢？先返回一个假IP并监听该IP，趁客户端的处理间隙将真的DNS解析好，假IP被访问的时候就将它映射到一个先前的请求，最后对外网的连接由mihomo完成，这样可以减小用户的等待时间。

{% mermaid %}
sequenceDiagram
  box 本地设备与代理
  participant CP as 代理客户端
  participant FIP as 假IP
  participant M as mihomo
  end
  box 公网
  participant PDNS as 公共DNS服务器 
  participant PS as 代理服务器
  participant WEB as 互联网
  end
  CP->>M: 代理DNS请求
  M-->>CP: 假DNS回复并记录
  M->>PDNS: DNS请求
  PDNS-->>M: DNS回复
  critical
    CP->>FIP: 使用假IP和代理协议访问
    FIP->>M: 监听并匹配
  option 分流直连
    M->>WEB: DIRECT直连
  option 分流代理
    M->>PS: PROXY代理
    PS->>WEB: 代理访问互联网
  end
{% endmermaid %}

`fake-ip`模式理论上很好，但是实际上有两处掣肘：

1. 对于延迟的提升几乎感受不到，特别是由于系统还有DNS缓存这种机制。
2. 只适用于代理模式和TUN模式

问题2就是我们遇到的问题。当我们在fake-ip模式+代理模式下将mihomo作为系统DNS使用时，对于没有设置或者不接受代理配置的软件会发生什么事情？

{% mermaid %}
sequenceDiagram
  box 本地设备与代理
  participant C as 非代理客户端
  participant FIP as 假IP
  participant M as mihomo
  end
  box 公网
  participant PDNS as 公共DNS服务器 
  participant PS as 代理服务器
  participant WEB as 互联网
  end
  C->>M: 普通DNS请求
  M-->>C: 假DNS回复并记录
  M->>PDNS: DNS请求
  PDNS-->>M: DNS回复
  C->>FIP: 使用假IP直接访问，被丢弃
{% endmermaid %}

可以得知，在没有使用代理协议访问FakeIP时，请求包无法被mihomo匹配，因此会被直接丢弃，自然就无法联网了。

由于我将KDE Plasma系统代理配置为了mihomo，并且firefox默认会读取Plasma的代理配置，因此浏览器内访问是正常的。但是对于把mihomo的DNS当普通DNS来用的软件，这就是灾难。

## 解决方案

很简单，把`fake-ip`改成`redir-host`即可。

```yaml /etc/mihomo/config.yaml
dns:
  enable: true
  prefer-h3: true
  listen: 127.0.0.1:5053
  ipv6: true
  use-hosts: true
  use-system-hosts: true
  enhanced-mode: redir-host
```

需要注意的是，改动之后，`dns`栏中配置的其他`fake-ip`相关配置都将失效。

另外一个方案就是改用TUN模式，但是TUN模式在复杂的网络环境下（我装了很多KVM虚拟机和Docker容器）容易出BUG，因此我就没用。