---
title: Linux安装Stable Diffusion WebUI为systemd服务
categories:
  - 软件
tags:
  - Linux
  - StableDiffusion
  - systemd
abbrlink: 57669
date: 2025-05-05 03:06:23
---

本文章介绍了在Ubuntu上使用Conda环境部署Stable Diffusion WebUI，并将其部署为systemd系统服务的方法。与此同时，还包含这些操作的要点。

Stable Diffusion WebUI（简称SD WebUI，下文简称SD）的本地部署一直是很热的话题，但是在Linux上的服务端部署一直是一个难题。不像在Windows上，我们有秋叶大佬开发的绘世启动器一键懒人包，在Linux上我们必须手动下载Stable Diffusion WebUI并进行配置。

## 为什么要用Linux？

首先显而易见的是，Linux下SD的性能浪费更少。我之前使用秋叶的整合包时，网页面板每次都需要缓冲半分钟才能打开，而使用Linux时几乎是秒开。在绘图性能上也是Linux略优，这在许多人的测试中均有体现。

其次，SD陷在CUDA生态里面，在Windows上使用A卡和I卡运行SD是使用DirectML兼容层转接的，存在比较明显的性能损耗。而在Linux上，A卡可以使用ROCm来运行SD，可以满血运行。

第三，SD的启动相当耗时，并且由于会预分配显存（将模型一次性搬进显存）导致显存爆满，显然不适合在显卡有多任务需求（例如左边打游戏，右边看视频）的日用机，否则你只能在每次跑图之前把软件全关了。

也因为如此，我的方案是在服务器上安装一块P104-100，运行SD，其它电脑用网页访问就能直接跑图了。

## 安装N卡驱动

Ubuntu默认使用的是开源的nouveau驱动，但是性能相对会弱一些。在GUI环境下，可以直接进入软件包更新=>附加驱动来切换到NVIDIA专有驱动（重启生效）。

如果你是其它的操作系统或者没有GUI环境，就需要自己手动去找方法切换过去了（或者可能可以忽视掉，直接装CUDA）。

## 安装miniconda3

在相当一部分系统中，python包都是系统级管理的（externally managed），直接使用pip安装会导致报错，提示需要创建venv（python虚拟环境）或直接使用系统包管理器（如apt）安装。venv不失为一个好方案，但是无法自由切换python版本（SD使用较老的python3.10.6，有相当大概率系统装的不是这个版本），限制还是相对比较大的。

conda是一个包和环境管理工具，能够创建高度独立的python环境，并且可以直接修改python版本，相比venv来说使用更为简便。在Windows上一般会直接使用其整合包Anaconda（近几年的信息课上一般都会使用它），由于包含了大量的额外包导致其较为臃肿，因此就有了其最小环境miniconda（没有GUI）。

{% note warning::由于新版本conda的shell激活用时极长，现已不推荐使用miniconda。可以改用Astral uv（支持自定义python与pip版本）等工具来进行虚拟环境管理。 %}

在Linux终端中逐行输入以下命令：

``` bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash ./Miniconda3-latest-Linux-x86_64.sh
```

这个命令会下载官方的安装脚本并执行它。中间会停下来几次，按照以下指南操作：

第一次停下来是提醒你查看使用条款，回车即可。

第二次停下来是让你接受使用条款，输入yes然后回车即可。

第三次停下来是让你确认安装路径，一般是位于你的用户文件夹中。回车即可。

安装完毕后，需要重启终端以完成安装（对于桌面Ubuntu系统，关闭命令窗口然后新建一个就行；对于SSH连接，需要退出并重连）。

重启之后会自动进入基础conda环境(base)，输入以下命令以禁用该功能：

``` bash
conda config --set auto_activate_base false
```

然后输入`conda deactivate`来退出基础环境。

最好再配置一下镜像源，此处就略过了。

## 下载并安装Stable Diffusion WebUI

先安装好git。不同的Linux系统上安装git的方法都不太一样，比如在Ubuntu和Debian上可以直接通过apt安装：

``` bash
sudo apt install git
```

然后找一块风水宝地准备下载SD。不建议放在`/etc`或者`/var`之类的地方，也不建议放在个人目录（`/home/{username}`，或者直接显示为~）下。我选择的是在根目录(/)下创建port-apps文件夹。

输入以下代码来拉取SD：

``` bash
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui
```

如果你没有能够访问GitHub的环境，可以使用代理站（不能保证永久可用，可以查看[Github Proxy 文件代理加速](https://github.akams.cn/)来寻找可用源）。

``` bash
git clone https://github.proxy.class3.fun/https://github.com/AUTOMATIC1111/stable-diffusion-webui
```

等待进度条走完，然后进入目录：

``` bash
cd stable-diffusion-webui
```

创建并进入conda环境：

``` bash
conda create -n sd-webui
conda activate sd-webui
```

安装依赖项：

``` bash
pip install -r requirements.txt
```

切换huggingface源为镜像站：

``` bash
export HF_ENDPOINT=https://hf-mirror.com
```

如果你是普通的N卡（不是P104这种把半精度阉割了的），直接运行以下命令即可开始运行：

``` bash
python ./launch.py --listen
```

皮衣刀客把某些矿卡阉割了半精度，此时需要禁用半精度，否则出图会很慢：

``` bash
python ./launch.py --listen --no-half --no-half-vae
```

### 为什么不直接运行webui.sh？

webui.sh是经过了高度整合的打包运行脚本，理论上来讲是完全没有问题的，但是当你遇到需要修改命令行参数、绕过huggingface被墙等问题时，整个脚本的修改会非常麻烦。这种情况下，其实还不如直接自己写一个脚本。并且，webui.sh直接使用python venv，出于文章开头的理由，这样显然没有使用conda好。

## 安装为系统服务

在服务器环境中，每次重启后都手动启动SD显然是不方便的，因此可以考虑将它配置为系统服务，这样需要的时候可以立刻开始出图。在Linux中，一般使用systemctl来管理系统服务，我们也可以自己编写配置单元文件来创建服务。服务单元文件位于`/etc/systemd/system/`下，以.service结尾。

SD的启动可以使用bash脚本进行，这是在Linux下最方便的方式。cd到SD目录中，输入如下指令创建脚本：

``` bash
vim launch.sh
```

按i开始编辑。

在bash脚本中，`conda activate`命令是要在source完conda的运行文件之后才能直接使用的：

``` bash
#!/bin/bash
source /home/{username}/miniconda3/etc/profile.d/conda.sh
conda activate {venv_name}
```

实际使用时，需要将以上代码中的`{username}`替换成安装conda的用户名，或者将source的文件路径改成你conda的安装目录下的conda.sh。`{venv_name}`也需要替换成你的conda环境名称，本例中是`sd-webui`。

``` bash
#!/bin/bash
source /home/{username}/miniconda3/etc/profile.d/conda.sh
conda activate sd-webui
#使用huggingface镜像源
export HF_ENDPOINT=https://hf-mirror.com
#调用python运行SD，允许在面板中安装插件，自动处理精度差异，允许非本地连接，开启xformers优化
#需要将路径调整为你自己的安装路径
python3 /port-apps/stable-diffusion-webui/launch.py --enable-insecure-extension-access --precision autocast --listen --xformers
```

先`bash ./launch.sh`测试一下，确保没报错后，开始编写配置单元文件：

``` bash
sudo vim /etc/systemd/system/sd-webui.service
```

按i开始编辑。

``` ini
[Unit]
#描述
Description=Stable Diffusion Webui Daemon
#前置服务，由于需要监听tcp栈，放一个network.target
After=network.target
 
[Service]
#直接管辖
Type=simple
#启动脚本
ExecStart=bash /port-apps/stable-diffusion-webui/launch.sh
#使用安装conda的用户运行服务
User=conda_user
#这一行很重要，不设置会导致显示最终成果时出现access denied错误
WorkingDirectory=/port-apps/stable-diffusion-webui
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
```

重载systemctl：

``` bash
sudo systemctl daemon-reload
```

尝试启动服务：

``` bash
sudo systemctl start sd-webui
```

如果之前直接运行脚本没有报错，则使用systemctl时也应该没有报错。执行`sudo systemctl enable sd-webui`来允许其开机自启。