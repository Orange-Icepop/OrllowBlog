---
title: 删除Grub的EFI BootNext启动项
categories:
  - 系统
tags:
  - UEFI
  - Grub
  - Linux
  - ArchLinux
abbrlink: a4e196f2
date: 2026-09-30 11:08:08
---

## 问题描述

前两天刚滚了一下ArchLinux，重启的时候发现Grub的启动项好像变长了不少。再重启一下，定睛一看，所有原来的启动项都多出了对应的一个新启动项，后面标着`(EFI BootNext)`，很是碍眼。

{% note info %}

## 啥是EFI BootNext？

`BootNext` 是 UEFI 规范中定义的一个标准变量，存储在主板 NVRAM 中。它的作用是指定下一次开机时优先启动的条目，仅生效一次。

具体流程是这样的：操作系统或引导程序向 `BootNext` 写入一个启动项编号（如 `Boot0001`）；下次开机时，UEFI 固件会读取这个变量，删除它，然后加载对应的启动项。这次启动结束后，系统就回到正常的 `BootOrder` 顺序，`BootNext` 已经不复存在。

## 为什么需要它？BitLocker 是关键

在双系统场景下，从 Linux 重启到 Windows 有一个经典难题。如果你在 GRUB 里直接 chainload Windows 的 bootmgfw.efi，TPM 的 PCR 测量值会发生变化，BitLocker 会认为启动环境异常，进而弹出恢复密钥提示。

而通过设置 BootNext 变量来启动 Windows，UEFI 固件会原生地引导 Windows Boot Manager，TPM 状态正常，BitLocker 就不会报警。这正是 Debian 社区专门提出 efibootnext 包请求的原因——提供一个 GRUB 菜单项，通过设置 BootNext 变量来安全地重启到 Windows。

不过对于非双系统用户来说这东西就是纯鸡肋了。

## GRUB 2.16 的行为变化

### 2.1 新增 `efibootnext` 模块

GRUB 2.16 新增了一个 **`efibootnext` 模块**，专门用于处理一次性 EFI 启动项。该模块依据 ESP 分区的 PARTUUID 和 vendor directory 来识别可用的 EFI 启动项。换言之，它的作用类似于主动探测。

### 2.2 `os-prober` 的扫描范围扩大

变化不只来自 `efibootnext` 模块。GRUB 2.16 还修改了 `30_os-prober` 脚本的行为：在支持 EFI 和 Legacy BIOS 双模式的 x86 系统上，`os-prober` 会**同时报告 EFI 和 BIOS 两种引导方式下的启动项**。

这两个改动叠加在一起，结果就是：GRUB 在生成菜单时会扫描并列出 ESP 分区上所有可用的 EFI 启动条目，包括当前不活跃的、旧的、甚至已删除系统的残留条目。这些多出来的“EFI BootNext”选项正是这些被扫描到的 UEFI 启动项在 GRUB 菜单中的呈现。

需要说明的是，BootNext 条目与 `os-prober` 的常规条目是两个独立的机制，只是它们在 GRUB 2.16 中同时被启用，所以会看到菜单里突然多出一批条目。

{% endnote %}

## 如何清理这些多余的选项

有两种思路：一是让 GRUB 不再生成这些菜单项，二是从 UEFI 层面彻底删除不需要的启动项。推荐先用第一种方法，如果菜单里仍然有残留，再用第二种。

### 方法一：通过 GRUB 配置禁用 BootNext 条目

这是最直接的方法。编辑 `/etc/default/grub`，在文件末尾添加一行：

```bash
GRUB_DISABLE_BOOTNEXT=true
```

然后重新生成 GRUB 配置。不同发行版的命令略有差异：

```bash
# Debian / Ubuntu / Arch
sudo grub-mkconfig -o /boot/grub/grub.cfg

# 或者
sudo update-grub
```

在 EndeavourOS 等 Arch 系发行版上，这个配置项已经被验证有效。CachyOS 社区也确认，GRUB 2.16 的 BootNext 条目可以通过 `GRUB_DISABLE_BOOTNEXT=true` 来禁用。

**需要注意的几个点：**

- 如果你同时需要保留 Windows 的 `os-prober` 条目，不要设置 `GRUB_DISABLE_OS_PROBER=true`，否则 Windows 启动项也会一起消失。
- 在 EndeavourOS 的讨论中，有用户将 `GRUB_DISABLE_BOOTNEXT=true` 和 `GRUB_DISABLE_UEFI_FIRMWARE=true` 一起添加。后者还会隐藏 UEFI 固件设置菜单项，视你的需要决定是否添加。
- 每次 GRUB 升级后，如果 `/etc/default/grub` 被覆盖（可能性比较小），可能需要重新添加这行配置。

### 方法二：用 `efibootmgr` 彻底删除残留条目

如果配置禁用后菜单里仍有残留，说明这些条目实际存在于 UEFI 的 NVRAM 中。此时需要用 `efibootmgr` 直接管理它们。

**第一步：查看当前所有 UEFI 启动项**

```bash
sudo efibootmgr -v
```

输出类似：

```
BootCurrent: 0001
Timeout: 10 seconds
BootOrder: 0000,0001,0004,9999
Boot0000* Windows Boot Manager
Boot0001* Linux Boot Manager
Boot0004* Internal Hard Disk
Boot9999* USB Drive (UEFI)
```

其中 `BootOrder` 后面的编号（如 `0000`、`0001`）就是启动项编号。

**第二步：删除不需要的条目**

假设要删除 `Boot0004`：

```bash
sudo efibootmgr -b 0004 -B
```

`-b` 指定启动项编号，`-B` 表示删除该条目，它也会从 `BootOrder` 中移除。

**第三步：确认结果**

再次运行 `sudo efibootmgr -v`，确认目标条目已消失。

**不要随意删除 `BootOrder` 中当前正在使用的条目**（即 `BootCurrent` 对应的编号），否则可能导致系统无法启动。

### 附录：用 `grub-editenv` 设置一次性启动

清理完菜单之后，如果仍然需要“从 Linux 重启到 Windows 且不触发 BitLocker”的功能，可以用 GRUB 自带的一次性启动机制：

```bash
sudo grub-editenv set next_entry <entry_id>
sudo reboot
```

其中 `<entry_id>` 是 GRUB 菜单中 Windows 条目的标识符（通常是 `osprober-efi-XXXXXXXX` 格式）。这相当于在 GRUB 层面手动设置了一次性启动，不影响正常菜单的简洁性。

## D老师小结

| 操作 | 命令/配置 |
|------|-----------|
| 禁用 BootNext 菜单条目 | `/etc/default/grub` 添加 `GRUB_DISABLE_BOOTNEXT=true` |
| 重新生成配置 | `sudo grub-mkconfig -o /boot/grub/grub.cfg` |
| 查看 UEFI 启动项 | `sudo efibootmgr -v` |
| 删除指定启动项 | `sudo efibootmgr -b <编号> -B` |
| 手动设置一次性启动 | `sudo grub-editenv set next_entry <ID>` |

GRUB 2.16 的这项改动本身是有价值的——它为安全的双系统切换提供了更好的支持。只是默认行为过于激进，把一堆用户并不需要的启动项都塞进了菜单。理解 `BootNext` 的机制之后，你就能根据自己的需要决定保留还是清理它们。
