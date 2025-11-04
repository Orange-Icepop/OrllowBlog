---
title: 解决英伟达旧显卡DP接口无POST和BIOS输出问题
date: 2025-02-11 21:55:30
categories:
  - 硬件
tags:
  - 图拉丁
---

也就是开机一路黑屏直到进系统。顺带还有开启UEFI后DVI接口输出开机图标被拉长的问题。

省流：显卡VBIOS的GOP版本过低导致的。

## 起因

前段时间整了一台x99的电脑（华南x99-TF，公版AMI BIOS），搭配Tesla M40显卡，打算跑服务器兼AI，整了一张几十块钱的Quadro K600做显示输出。然后发现功耗和噪音都太高了，就准备直接变成主力机。

{% note info::在遇到这个问题之前，也出现过症状一样的问题，不一样的是DVI接口也发病，最后打开了安全启动以同时关掉CSM和打开UEFI解决的问题。可以看这篇文章：[解决英伟达旧显卡开启UEFI后无POST和BIOS输出问题](/2025/02/11/解决英伟达旧显卡开启UEFI后不显示POST与BIOS画面的问题/)。 %}

由于卧室里的显示器只有一个HDMI，剩下的只有一个DP和一个Mini DP，刚好K600上面的2号口也是DP，所以就淘了一根拆机DP线，插上，开机，问题复现。

## 试过的方案

以下方案不一定有用，但是万一其中哪个是前置条件呢？

- 打英伟达官方的[DP补丁](https://www.nvidia.com/en-us/drivers/nv-uefi-update-x64/)
- 上[TechPowerUp](https://www.techpowerup.com/vgabios/)找最新的当前显卡的VBIOS，用[NVFLASH](https://www.techpowerup.com/download/nvidia-nvflash/)刷进去

## 最终解决方案

问了一趟DeepSeek，我把情况都说明白之后，它根据我是纯UEFI环境的线索直接扒拉出来两种可能性，第一种是显卡初始化优先级问题，说白了就是把M40当主显卡用了，但是使用DVI的时候这个问题已经解决，直接Pass掉。

还有一种可能性，就是UEFI GOP（Graphics Output Protocol）版本太低导致的。

{% note info::UEFI GOP（图形输出协议）是UEFI固件的一部分，用于在系统启动时初始化和控制图形硬件。它提供了一种标准化的方法来处理图形输出，使得不同的硬件和操作系统可以更好地兼容和协作。 %}

查了一些案例，好像这个GOP的问题确实有些道理。那就试试。

先用GPU-Z备份VBIOS。进入软件页面后，在界面最下方的下拉栏里选中K600，然后左击显卡图标左下方的那个有点像分享按钮的东西，确认。备份期间显示屏会闪黑，这是正常情况。然后指定导出文件路径，导到顺手的地方就行，桌面上都可以（前提是你有随手清理的习惯）。

接下来的操作是重点。下载GOPUpd工具并解压出来（链接会稍后贴上），顾名思义这个工具就是用来更新GOP版本的。需要注意的是，该工具仅支持Windows且依赖Python环境，因此需要预先安装Python。确认有没有装Python只需要打开命令行（win+R键，输入cmd，回车），输入如下命令：

``` cmd cmd
python --version
```

有返回Python几点几几的就说明安装好了。

没有安装也不要紧，微软商城里搜索Python，安装最新的即可。

然后，把备份出来的VBIOS文件(.rom)拖到GOPupd.bat上，程序会自动解压VBIOS并弹出操作窗口。正常情况下只要键入一个y并回车就行，程序会把新的.rom文件放在程序同目录下，文件名后面会添加_updGOP方便辨识。

不查不知道，一查吓一跳，原卡的GOP版本只有0x1000D，而最新的版本已经更新到0x10038了。这还是十六进制的版本号，相当于隔了43个版本，也难怪有问题了。

## 刷入VBIOS

接下来就是刷入VBIOS了，AMD显卡的方法可以看参考文章，此处只讲解使用NVFLASH刷N卡的方法。

下载好NVFLASH之后，解压，将之前GOPUpd工具制作出来的.rom文件拖到NVFLASH.exe的同一目录下。

然后就是在此处打开命令行。Windows11用户可以直接右击NVFLASH当前所在目录的空白区域选择打开终端，其它版本的用户可以像之前那样打开cmd，然后使用cd [路径]命令导航到NVFLASH所在目录。

然后，输入以下命令：

``` cmd cmd
.\nvflash --protectoff
```

这个命令是用于解除VBIOS的写保护的，在刷完之后它会自己恢复。

接下来，输入如下命令开始刷入VBIOS。在此期间，切记不要关机或是断电，否则显卡会由于VBIOS不完整而无法显示，俗称变砖。

``` cmd cmd
.\nvflash -6 [VBIOS文件名]
```

其中的\[VBIOS文件名\]填写新的VBIOS文件名。然后，输入y，回车，等待大约一分钟，显示”A reboot is required for the update to take effect”，VBIOS就刷好了。

插上DP线，重启，自检通过，亮图标，按del，进BIOS，一气呵成。

这个问题的解决还带了一个意想不到的副作用，就是开启UEFI后DP接口输出的视频信号的主板厂商图标被拉长的问题得到了解决。（终于不用看那面筋人一样的开机图标了ヾ(≧▽≦*)o）

## 参考文档

[卡灯黑屏无信号 UEFI显卡玄学故障全解_显卡_什么值得买](https://post.smzdm.com/p/a4xo9438/)