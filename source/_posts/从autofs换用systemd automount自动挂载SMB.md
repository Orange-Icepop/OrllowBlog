---
title: 从autofs换用systemd automount自动挂载SMB
categories:
  - 软件
tags:
  - Linux
  - systemd
  - SMB
abbrlink: b44a064
date: 2026-10-17 05:25:22
---

在过去的很长一段时间内，当Linux用户需要自动按需挂载远程的SMB到本地路径时，autofs都是首选。我也不例外，在我的ArchLinux上使用autofs来执行这个操作。然而，随着时间的推移，我越来越觉得autofs落后并且BUG频出（包括会阻塞式卡死Dolphin），特别是这玩意儿还在AUR里，更新起来非常麻烦。

于是乎问了一下D老师，告诉我其实systemd自带功能automount就能解决（systemd的恩情还不完\o/）。真是...绕了好大一圈啊...

## 配置步骤

### 第一步：提供凭据

```ini /etc/samba/nas-credentials
username=<用户名>
password=<密码>
domain=<工作组，此项可选>
```

文件名自选。

然后要把凭据文件的所有者变成root，权限改成600，否则cifs可能拒绝使用。

``` bash
sudo chown root:root /etc/samba/nas-credentials #如果你写凭据文件的时候没有用root的话
sudo chmod 600 /etc/samba/nas-credentials
```

### 第二步：配置systemd单元

配置automount需要两个单元文件，一个 `.mount` 单元和一个 `.automount` 单元。单元文件的命名必须与挂载点路径对应，例如，挂载点为 `/mnt/nas`，则单元名应为 `mnt-nas.mount` 和 `mnt-nas.automount`。

{% note warning %}
如果路径里本身有 `-`，需要转义成 `\x2d`。比如 `/mnt/my-nas` 对应 `mnt-my\x2dnas.mount`。
{% endnote %}

以下为示例文件。`Description` `What` `Where` 等参数与文件名均需按照你自己的情况填写。

```ini /etc/systemd/system/mnt-nas.mount
[Unit]
Description=SMB Share //192.168.1.100/share
After=network-online.target
Wants=network-online.target

[Mount]
What=//192.168.1.100/share
Where=/mnt/nas
Type=cifs
Options=credentials=/etc/samba/nas-credentials,uid=1000,gid=1000,vers=3.0,_netdev,rsize=1048576,wsize=1048576
```

```ini /etc/systemd/system/mnt-nas.automount
[Unit]
Description=Automount for SMB Share

[Automount]
Where=/mnt/nas
TimeoutIdleSec=5min

[Install]
WantedBy=multi-user.target
```

{% note warning %}
`.automount` 里不要写任何 `After=`、`Wants=` 之类的网络依赖，网络依赖只放在 `.mount` 里。我一开始两边都写了 `After=network-online.target`，结果开机直接 ordering cycle，一堆 skip，NetworkManager 都没起来。
{% endnote %}

### 第三步：重载并启动自动挂载

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now /etc/systemd/system/mnt-nas.automount
```

只 `enable .automount`，不要 `enable .mount`。enable 了 `.mount` 就等于开机直接挂载，按需的意义就没了。

顺带一提：在启用 systemd 方案之前，记得把 autofs 彻底停掉：`systemctl stop autofs`，`disable`，`umount -l /mnt/nas`，然后把 AUR 的包 `pacman -R` 掉。不然两个东西抢同一个挂载点，又会多出一堆莫名其妙的报错。

至此，当访问挂载点时，systemd就会替代autofs执行按需自动挂载。