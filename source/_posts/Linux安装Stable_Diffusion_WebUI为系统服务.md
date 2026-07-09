---
title: Linux部署Stable Diffusion WebUI为系统服务（uv/Conda）
categories:
  - 软件
tags:
  - Linux
  - StableDiffusion
  - systemd
abbrlink: d57de0fb
date: 2025-05-05 03:06:23
---

本文介绍了在Ubuntu上部署Stable Diffusion WebUI，并将其配置为systemd系统服务的方法。推荐使用Astral uv作为虚拟环境管理工具，同时也会提供conda方案作为备选。

Stable Diffusion WebUI（简称SD WebUI，下文简称SD）的本地部署一直是很热的话题，但在Linux上的服务端部署一直是一个难题。不像在Windows上有绘世启动器这样的懒人包，在Linux上我们必须手动下载并进行配置。

{% note info %}

（2026/7/9更新）

Stable Diffusion WebUI的master branch已经两年没有更新，依赖链出现严重损坏。请使用dev分支，或者如果愿意学习“工作流”玩法的话（其实也没有复杂多少，而且扩展性强得多），可以改用[ComfyUI](https://docs.comfy.org/zh)。

{% endnote %}

## 为什么要用Linux？

首先显而易见的是，Linux下SD的性能浪费更少。我之前使用秋叶的整合包时，网页面板每次都需要缓冲半分钟才能打开，而使用Linux时几乎是秒开。在绘图性能上也是Linux略优，这在许多人的测试中均有体现。

其次，SD陷在CUDA生态里面，在Windows上使用A卡和I卡运行SD是使用DirectML兼容层转接的，存在比较明显的性能损耗。而在Linux上，A卡可以使用ROCm来运行SD，可以满血运行。

第三，SD的启动相当耗时，并且由于会预分配显存（将模型一次性搬进显存）导致显存爆满，显然不适合在显卡有多任务需求（例如左边打游戏，右边看视频）的日用机，否则你只能在每次跑图之前把软件全关了。

也因为如此，我的方案是在服务器上安装一块P104-100，运行SD，其它电脑用网页访问就能直接跑图了。

## 前置准备

### 安装N卡驱动

Ubuntu默认使用开源的nouveau驱动，无法使用CUDA加速。在GUI环境下，可以进入**软件包更新 → 附加驱动**来切换到NVIDIA专有驱动（重启生效）。如果你的环境没有GUI，则需要使用包管理器切换到专有驱动。相关方法请自行查询。

### 安装git

``` shell
sudo apt install git
```

## 安装隔离环境

在相当一部分系统中，python包都是系统级管理的（externally managed），直接使用pip安装会导致报错，提示需要创建venv或直接使用系统包管理器安装。python venv无法自由切换python版本（SD使用较老的python3.11），限制较大；系统包管理器则有依赖地狱的问题。我们可以使用其他虚拟环境工具来解决。

{% note warning %}
虽然Anaconda/miniconda历来都被诸多教程推崇，但是由于新版本conda的shell激活用时极长，现已不推荐使用。推荐改用Astral uv（支持自定义python与pip版本）等工具。下面将同时提供两个方案的实现，首选uv。
{% endnote %}

{% tabs %}

<!-- tab uv（推荐） -->

uv是一个快速、轻量、基于Apache-2.0开源的python虚拟环境和项目管理工具，使用Rust重写了pip，在包安装时有着巨大的速度提升，并且提供了很好的显式管理命令。

请优先使用系统级包管理器安装。如果没有，才[使用官方安装脚本安装](https://docs.astral.sh/uv/getting-started/installation/)。

uv无需额外配置即可使用，但建议配置镜像源。此配置[参考知乎](https://zhuanlan.zhihu.com/p/1971030991746368494)。

``` toml
# ~/.config/uv/uv.toml

# CPython代理配置，用于uv python install下载Python解释器
python-install-mirror = "https://ghfast.top/https://github.com/astral-sh/python-build-standalone/releases/download"

# PyPI源配置
[[index]]
name = "ustc"
url = "https://mirrors.ustc.edu.cn/pypi/simple" 
```

<!-- endtab -->

<!-- tab miniconda -->

conda是一个包和环境管理工具，能够创建高度独立的开发环境，并且可以直接修改python版本。

{% note warning %}
Anaconda与miniconda均为Anaconda公司运营的私有软件，若在企业网络环境下使用，你所在的公司可能收到律师函。
{% endnote %}

优先使用系统级包管理器安装。如果没有，再手动安装：

``` shell
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash ./Miniconda3-latest-Linux-x86_64.sh
```

按提示完成安装后，重启终端。禁用默认激活基础环境：

``` shell
conda config --set auto_activate_base false
conda deactivate
```

建议再配置镜像源，此处略过。

<!-- endtab -->

{% endtabs %}

## 下载并安装Stable Diffusion WebUI

找一块风水宝地准备下载SD。不建议放在`/etc`或`/var`下，也不建议放在个人目录下。我选择使用`/opt`。

``` shell
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui
```

如果没有访问GitHub的环境，可以使用代理站（可查看[Github Proxy 文件代理加速](https://github.akams.cn/)寻找可用源）：

``` shell
git clone https://github.proxy.class3.fun/https://github.com/AUTOMATIC1111/stable-diffusion-webui
```

进入目录：

``` shell
cd stable-diffusion-webui
```

{% note info %}
uv有两种用法：一是使用uv开头的自有命令（无需手动激活环境）；二是将uv当作虚拟环境管理工具使用（需手动source）。前者更稳定且能使用uv的特有优化，后者兼容性更好但生产环境不建议。切勿混用。
{% endnote %}

### 安装依赖

{% tabs %}

<!-- tab 纯uv -->

``` shell
uv init --python 3.11
uv add -r requirements.txt
```

<!-- endtab -->

<!-- tab uv虚拟环境 -->

``` shell
uv venv --python 3.11
source ./.venv/bin/activate
uv pip install pip
pip install -r requirements.txt
```

<!-- endtab -->

<!-- tab miniconda -->

``` shell
conda create -n sd-webui python=3.11
conda activate sd-webui
pip install -r requirements.txt
```

<!-- endtab -->

{% endtabs %}

### 配置镜像源并测试运行

切换huggingface源为镜像站：

``` shell
export HF_ENDPOINT=https://hf-mirror.com
```

运行SD WebUI：

``` shell
# 纯uv
uv run ./launch.py --listen
# 虚拟环境 / conda
python3 ./launch.py --listen
```

如果是被皮衣刀客阉割了半精度的矿卡（如P104），需额外加上`--no-half --no-half-vae`参数。

## 安装为系统服务

在服务器环境中，每次重启后都手动启动SD显然不方便。我们可以使用systemd将其配置为系统服务。

{% note info %}

### 为什么不直接运行webui.sh？

webui.sh是高度整合的打包运行脚本，本身没有问题，但当需要修改命令行参数、绕过huggingface被墙等问题时，修改这个脚本非常麻烦。并且webui.sh直接使用python venv，无法自由切换python版本，不如uv或conda灵活。

{% endnote %}

### 编写启动脚本

cd到SD目录，创建启动脚本：

``` shell
vim launch.sh
```

{% tabs %}

<!-- tab 纯uv -->

超简单的！只要systemctl里的`WorkingDirectory`设置正确，这么写就行了。

``` shell
#!/bin/bash
export HF_ENDPOINT=https://hf-mirror.com
/usr/bin/uv run ./launch.py --自定义参数
```

<!-- endtab -->

<!-- tab uv虚拟环境 -->

需要先source虚拟环境文件。

``` shell
#!/bin/bash
source ./.venv/bin/activate
export HF_ENDPOINT=https://hf-mirror.com
python3 ./launch.py --自定义参数
```

<!-- endtab -->

<!-- tab miniconda -->

``` shell
#!/bin/bash
source /home/{username}/miniconda3/etc/profile.d/conda.sh
conda activate sd-webui
export HF_ENDPOINT=https://hf-mirror.com
python3 ./launch.py --自定义参数
```

将`{username}`替换为你的用户名。

<!-- endtab -->

{% endtabs %}

我使用的自定义参数是`--enable-insecure-extension-access --precision autocast --listen --xformers`，允许在面板中安装插件，自动处理精度差异，允许非本地连接，开启xformers优化。

先`bash ./launch.sh`测试，确保没报错。

### 编写systemd单元文件

``` shell
sudo vim /etc/systemd/system/sd-webui.service
```

``` ini
[Unit]
Description=Stable Diffusion Webui Daemon
#前置服务，由于需要监听tcp栈，放一个network.target
After=network.target

[Service]
Type=simple
ExecStart=bash /port-apps/stable-diffusion-webui/launch.sh
#这一行很重要，不设置会导致显示最终成果时出现access denied错误
WorkingDirectory=/port-apps/stable-diffusion-webui
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

重载并启动：

``` shell
sudo systemctl daemon-reload
sudo systemctl enable --now sd-webui
```

### 验证服务

``` shell
systemctl status sd-webui
```

确保状态为 `active (running)`。SD WebUI默认监听端口 `7860`，可以通过 `http://服务器IP:7860` 访问。

查看实时日志：

``` shell
journalctl -u sd-webui -f
```

---

至此，你已成功在Linux上部署Stable Diffusion WebUI并配置为systemd系统服务，可以随时通过网页远程使用GPU进行AI绘图。