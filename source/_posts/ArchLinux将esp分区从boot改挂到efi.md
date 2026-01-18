---
title: ArchLinux将ESP分区从/boot改挂到/efi
date: 2026-01-17T18:13:11+08:00
categories: [系统]
tags: [Linux, ArchLinux, UEFI]
---

btrfs，一个对于（几乎）所有ArchLinux用户都是救命稻草的东西，能够在滚动更新的不确定性中提供一个全身而退的安全出口。然而，如果你的ESP(EFI)分区没挂对，一旦回滚内核更新，系统就真的炸了......只能靠Arch Live CD来救命。

## 注意事项

**本文适用前提：你的Linux系统盘用的是btrfs，并且引导模式是UEFI。**

如果系统盘文件系统是ext4就没有必要关注了，它没有回滚的功能，也就不用操心这事儿。

如果你的电脑由于比较老或者你故意的，用的是传统非UEFI模式启动，那也不能改，滚内核的时候手动备份内核文件吧，否则刚改完你就发现你系统整个寄了。不过真的有人在Legacy模式下成功引导过btrfs文件系统的Linux吗......

## 错误示范

如果你的ESP分区挂到`/boot`了，那就是中招了。

终端里面`lsblk`一下，找到你的系统物理盘。有一个很小的分区，大小约在400MB至1GB左右，一般排在你系统卷的前面。这就是ESP(EFI)分区。在这行的右边会显示这个分区的挂载点，如果显示`/efi`或者`/boot/efi`（不建议，但不影响回滚，就别去动它了）就是没问题的；如果直接一个明晃晃的`/boot`写在那里，那就是我们这里要解决的问题。

{% folding cyan open::关于ESP分区 %}

在UEFI之前，传统的BIOS+MBR模式直接把操作系统引导程序的加载器（PBR）写进MBR（主引导记录）里，简单是简单，但是MBR本身就多大点地儿，存分区表还不够呢。位置也很尴尬，PBR和分区表混在一起，引导程序和操作系统文件混在一起，一点都不符合解耦原则。并且，一旦文件系统麻烦一点（比如btrfs这种有子卷，需要额外驱动的），MBR就寄了，只能指望grub2给你在`/boot`（FAT32）里存了个btrfs驱动，就算能成，稳定性也拉垮的一批，用起来不是一般的糟心。

PS.在Legacy环境下能够“免boot分区”启动NTFS文件系统上的Windows，实际上是因为微软走了个狗屎运，PBR的空间刚好能塞下一个最小的NTFS只读驱动，但是很显然这不是长久之计。要是当时微软没那么走运，现在也不会有那么多人骂ESP分区碍事了。

```mermaid
graph LR
    A[开机] --> B[BIOS自检]
    B --> C[找硬盘设备]
    C --> D[解析MBR]
    D --> E[执行PBR]
    E --> F[启动grub/bootmgr]
    F --> G[进入系统]
```

UEFI就做得不错了，它的层次分的很清楚：主板上的NVRAM存储启动配置，UEFI直接根据这个索引找到ESP分区，加载里面的`.efi`引导程序文件，然后引导程序文件负责解析文件系统找到内核。由于ESP分区大小可调并且相比PBR而言非常充裕，因此`.efi`文件可以内置大部分文件系统的完整驱动，不再需要费尽心思砍功能来塞进PBR里了。

```mermaid
graph LR
    A[开机] --> B[UEFI最小化自检]
    B --> C[访问NVRAM]
    C --> D[直接打开ESP]
    D --> E[执行efi文件]
    E --> F[进入系统]
```

当然，BIOS的硬件限制还是摆在那里，ESP分区的格式必须为FAT32，并且不可修改为别的文件系统，否则UEFI固件无法读取它。不排除某些厂商往BIOS里额外塞驱动，但是总归没人往里面塞ext4或btrfs的驱动。

ArchLinux不管内核是什么版本，`/boot`里的`initramfs`镜像和内核文件的名称都是不变的，这使得通过grub来启动任意快照的Linux而不修改ESP分区内容变得可能。

{% endfolding %}

{% noteblock warning::避坑 %}

没有恶意地讲一下，本人一开始是照着以下这个视频来装Arch的：

{% link 「Archlinux究极指南2025」从手动安装到显卡直通，最后删除Linux::https://www.bilibili.com/video/BV1L2gxzVEgs/ %}

这个视频现在看来有诸多错误，最严重的就是选择将ESP分区挂载到`/boot`。原作者已经发布了正确安装的视频，并且在评论区置顶了新视频。但是他没有删除旧视频，而旧视频仍然在B站`ArchLinux`搜索结果的第一位；与此同时，新视频更多的是“Linux上手过程指南”，而与安装Arch没有强相关，因此用户很容易掉进陷阱。

{% endnoteblock %}

## 迁移教程

### 准备ArchLinux Live CD

很显然，这种涉及系统启动项的设置是不能在系统内进行的。为此，我们就要请出我们的Live CD。具体怎么做就不细说了，实际上就是ArchLinux的安装盘。

### 1.挂载

btrfs由于全都是子卷，挂载相比ext4等要麻烦许多。

首先要找到你的系统盘。输入`lsblk`，找到你的系统盘和系统分区。系统盘应该有一个400M到1G大小（总之比较小）的分区，余下的空间是另外一个分区。前者是ESP分区，后者就是btrfs系统分区了。

假设ESP分区是`nvme0n1p1`，系统分区是`nvme0n1p2`：

```bash
# 挂载系统分区
mount -o subvol=@ /dev/nvme0n1p2 /mnt
# 挂载ESP分区到正确的位置
mount --mkdir /dev/nvme0n1p1 /mnt/efi
```

注：`mount --mkdir`参数是用于自动创建目标目录的，你也可以手动创建`/mnt/efi`目录。

### 2.拷贝

由于我们的内核文件之类的东西还留在ESP分区里，因此要先把它拷回到`/boot`。

```bash
cp -a /mnt/efi/* /mnt/boot/
```

先别急着清理，等到时候重新进入系统之后看看有没有翻车再搞。

### 3.改fstab

```bash
vim /mnt/etc/fstab
```

vim用不惯可以用nano或者vi，总之只要是文本编辑器就行。

打开之后，找到里面（应该是）唯一一处的`/boot`，改成`/efi`即可。保存并退出。

### 4.chroot、重建initramfs与grub

```bash
arch-chroot /mnt
# 重新生成initramfs，确保其指向新的/boot路径
mkinitcpio -P
# 重新安装并配置GRUB
grub-install --target=x86_64-efi --efi-directory=/efi
# 生成GRUB配置文件
grub-mkconfig -o /boot/grub/grub.cfg
```

等待重建完成，应该就没什么大问题了。

```bash
# 退出chroot环境
exit
# 卸载系统盘
umount -R /mnt
# 重启
reboot
```

### 5.收尾工作

当你重启之后，如果进入了系统，一切正常，那就应该没什么问题了。

第一步先打个快照，因为当前的内核还没有进入快照的保护范围。打快照的习惯因人而异，我的习惯是Syu之前打一次，Syu之后确定没问题了就再打一次，这样可以保留两次Syu之间的工作。

然后，你就可以开始清理了。你可以删掉`/boot`下的`EFI`文件夹，以及`/efi`下**除了EFI文件夹以外**的所有东西。

PS.我这里还有一个`BackupSbb.bin`文件，是一个UEFI固件恢复文件，这个最好还是留着。

搞完后再重启，如果没有问题就说明大功告成了。

## 局限

在你按照上面的指导完成修改后，需要注意，**它只是让你可以安全地回滚内核了。**对于整个系统核心组件彻底滚炸（连tty都进不了）的情况，由于无法调用btrfs的操作命令，仍然需要依赖Arch Live CD来进行回滚操作。

如果你不希望随身带一个Arch Live CD，可以使用`snap-pac`，`grub-btrfs`，`snap-pac-grub`这一套工具链，它通过`pacman钩子`来自动更新grub里的快照启动项。或者，直接换用`rEFInd`作为引导程序来自动查找btrfs子卷里的Linux。详情可以去查看[ArchLinux中文维基](https://wiki.archlinuxcn.org/wiki/Snapper#%E6%8F%90%E7%A4%BA%E4%B8%8E%E6%8A%80%E5%B7%A7)的相关说明。它也提供了关于建议的子卷配置和非只读快照的建议。

{% note warning::注意：这么做也需要将ESP分区改挂到`/efi`以让内核也能够被快照。 %}