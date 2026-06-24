---
title: 解决ArchLinux上Onlyoffice和WPS卸载后没有清除mimetype的问题
categories:
  - 软件
tags:
  - ArchLinux
  - mimetype
  - WPS
  - ONLYOFFICE
abbrlink: 19eb7291
date: 2025-12-27 01:45:45
---

## 症状

在尝试Linux上哪个办公软件最适合我时，我一度从AUR下载了`onlyoffice-bin`和`wps-office`两个包。后来我意识到还是Calligra和LibreOffice最好用，于是又把它们卸载了。然而，这玩意儿并没有完全卸载干净。

比如，如果右键某个docx文件，选择打开方式，有一个没有图标的ONLYOFFICE留在里面，如果尝试用这个东西打开，会显示类似“找不到/usr/bin/onlyoffice-desktopeditor”这样的提示。

WPS的问题相对更隐蔽，它是直接为自己的能够打开的所有文件类型都重新注册了一个mimetype，我是在KDE设置中的文件关联设置中搜了wps才发现的。

两个问题都无法通过重新安装包再`-Rns`来删除。

## 分析

不像Windows，Linux没有注册表这么复杂的东西。Linux桌面对于mimetype的处理很简单，它通过读取`/usr/share/mime/`和`~/.local/share/mime/`目录中的XML文件来创建新的MIME类型，随后通过扫描`/usr/share/applications/`与`~/.local/share/applications`下的Desktop文件来注册文件类型对应的候选应用程序（也就是选择使用什么软件打开的那个列表）。更详细的可以参见[ArchWiki](https://wiki.archlinuxcn.org/wiki/XDG_MIME_%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F)。

考虑到`/usr/share/`下面的那几个配置一般不会去改它，嫌疑最大的就是`~/.local/share/`下面的了。

## 解决

### WPS

WPS注册的MIME类型的XML配置在`~/.local/share/mime/packages/Override.xml`（WPS你还妄图通过用一个没有任何自己信息的文件名来糊弄过去，但是你不知道你用PascalCase本身在Linux里就很显眼吗？）。当然，WPS创建的文件可能不一样，还是得自己打开查找一下。

删除完该文件之后，我们还要重新建立一下MIME数据库：`update-mime-database ~/.local/share/mime`

{% note info %}

## 快速在一堆文件里找特定内容

当遇到像我们现在这样需要在一堆文件里搜索指定字符串时，一个个文件打开搜索很显然是不现实的。`grep`命令提供了一个非常好用的方法：`-r`参数。使用`grep -r 要搜索的字符串`，可以从当前目录开始向下递归搜索每一个文件，看看有没有你要检索的关键词。它还会自动跳过二进制文件，相当智能。

{% endnote %}

### ONLYOFFICE

ONLYOFFICE没有删除的是文档的处理程序注册信息，该文件位于`~/.local/share/applications/onlyoffice-desktopeditors.desktop`。（嗯，起码比WPS良心）

删除该文件之后，更新桌面数据库：`update-desktop-database ~/.local/share/applications/`

然后再去看打开方式菜单，ONLYOFFICE就应该消失了。