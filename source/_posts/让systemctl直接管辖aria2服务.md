---
title: 让systemctl直接管辖aria2服务
date: 2025-02-13T13:25:32+08:00
categories: 
  - 软件
tags: 
  - Linux
  - Aria2
  - systemctl
---

这篇省不了流，省了你也不知道怎么解决。

前段时间开始玩Ubuntu当服务器用，用apt装了个Aria2，又装了个AriaNG拿来挂下载。

然后问题来了，Aria2单文件下载线程数限死在16线程，相比Motrix慢如狗。然后就是扒拉下来Aria2源文件改了自己编译，又摸不清gcc咋编译……

最后找到了这个才解决：[P3TERX/Aria2-Pro-Core: Aria2 static binaries for GNU/Linux with some powerful feature patches. | 破解无限线程 防掉线程优化 静态编译 二进制文件 增强版](https://github.com/P3TERX/Aria2-Pro-Core)

把编译好的文件丢到/usr/bin下面，顺带按照教程放了个SysV脚本到/etc/init.d/下面让它自己生成服务配置文件。

然后问题又双叒叕来了，这个Aria2三天两头崩溃，systemctl就给你挂个running(exited)也不重启，手动重启服务又连不上，只能先stop再run，烦死个人。

所以最后就决定换成原生systemctl服务，直接管辖aria2c。

## 方案

以下代码由DeepSeek辅助生成，经本人验证可以正常使用。

### 一、创建标准systemd服务文件

``` bash
sudo vim /etc/systemd/system/aria2.service
```

（题外话：vim用习惯之后真的觉得nano就是史，一堆令人难受的快捷键，远不如vim好用）

按i切换到编辑模式，添加以下内容（内容可以自己调整）：

``` ini
[Unit]
Description=Aria2c Download Service
After=network-online.target
 
[Service]
User=aria2user  # 建议使用专用用户，然后和root放在同一组里。当然，也可以直接用root。
Group=aria2user # 用户组，单纯用root的话可以删掉
Type=simple     # 直接管辖
ExecStart=/usr/bin/aria2c --conf-path=/etc/aria2/aria2.conf # 这里的conf-path参数指的是aria2默认配置文件的路径，一般都在这个地方
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure # 让aria2服务在异常退出（非0状态码）的情况下自动重启。想要高稳定性的可以设成always。
RestartSec=3 # 3秒自动重启
KillSignal=SIGTERM
StandardOutput=syslog # 记录日志到系统日志
StandardError=syslog
SyslogIdentifier=aria2c
 
[Install]
WantedBy=multi-user.target
```

按Esc退出编辑模式，输入{% span code:::wq %}（有个冒号别漏了）保存并退出。

### 二、替换原有SysV文件

先停掉aria2服务：

``` bash
sudo systemctl stop aria2c
sudo systemctl disable aria2c
```

然后删除掉原有的SysV脚本：

``` bash
sudo rm /etc/init.d/aria2c 
```

或者你也可以先备份到别的地方（例如用户主目录）避免褒姒：

``` bash
sudo mv /etc/init.d/aria2c ~
```

然后重载系统服务，应用新配置：

``` bash
sudo systemctl daemon-reload
sudo systemctl enable --now aria2
```

最后验证确实在运行：

``` bash
systemctl status aria2
```

{% asset_img aria.webp aria2运行成功 %}

AriaNG也连好了，完美！