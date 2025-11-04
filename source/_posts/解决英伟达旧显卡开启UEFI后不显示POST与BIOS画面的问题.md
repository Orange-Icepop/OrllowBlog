---
title: 解决英伟达旧显卡开启UEFI后不显示POST与BIOS画面的问题
date: 2025-02-11 16:39:46
categories:
  - 硬件
tags:
  - 图拉丁
---

也就是开机一路黑屏直到进系统。

省流：打开安全启动。

## 起因

前段时间想玩E5和AI，整了一块华南x99-TF，搭配Tesla M40显卡（没错这货带Above 4G Decoding），又买了一张几十块钱的Quadro K600做显示输出。

显卡插好，开机，按Del开BIOS，按照教程切到Advanced，进入PCI Subsystem Settings，打开Above 4G Decoding，返回，进入CSM Configuration，关掉CSM Support。问题在这儿出现了，显示无法关闭，需要先把UEFI Video打开。

那行，打开UEFI Video，重启，按Del，直接黑屏。

摸黑按Esc，Enter，正常重启了，成功进入系统，M40也认到了，就是开机图标没了，BIOS也进不去。

## 试过的方案

先试了打英伟达官方的[DP补丁](https://www.nvidia.com/en-us/drivers/nv-uefi-update-x64/)，没用。

问了一圈说是显卡不支持UEFI启动，打开[GPU-Z](https://www.techpowerup.com/gpuz/)一看，这不是支持的吗？
以防万一上TechPowerUp找了最新的[VBIOS](https://www.techpowerup.com/vgabios/)，用[NVFLASH](https://www.techpowerup.com/download/nvidia-nvflash/)刷进去，然并卵。

然后就是反复重置BIOS，换着顺序开选项，也没有用。有一次还盲操摸到CSM选项，然而好像又被什么弹窗拦下来了。

## 最终解决

问题拖了一段时间，恰巧给这机子上了个Windows11要开安全启动和TPM2.0。TPM2.0据说华南这板子没给，那就先强制装上Windows11，然后重置BIOS去开安全启动。

打开BIOS，切到Security-Secure Boot Menu，将Secure Boot调成Enabled。咱也不知道Secure Boot Mode咋调，就把这个选项从Custom调成了Standard。

再把Above 4G Decoding打开，顺便回去看了一下CSM咋样。？怎么就没选项了？CSM Modue is not loaded due to active Secure Boot. 哦，安全启动顺便把CSM干掉了，UEFI估计也自动启动了。那就F10保存并退出得了。

然后是见证奇迹的时刻：我们的开机图标重新亮起来辣！BIOS也进得去了！

估计就是安全启动一开，调成了Standard之后，自动配置不仅省了手动读密钥的功夫，还直接把UEFI打开又把CSM干没了。真是一石二鸟耶。

唯一美中不足的就是开机图标被拉长了，看得我这个强迫症有点不舒服。但好歹BIOS不会需要经常重置了。