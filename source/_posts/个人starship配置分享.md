---
title: 个人starship配置分享
categories:
  - 系统
tags:
  - Linux
  - 命令行
abbrlink: 55757
date: 2026-05-15 23:10:53
---

## 效果展示

- 显示完整的工作目录（home目录自动折叠）
- 异常退出错误码显示，兼容管道
- 用户名显示
- Catppuccin Mocha配色方案
- git与技术栈识别

{% asset_img effect.webp 效果展示 %}

此方案在distrobox中也可以使用！并且会自动在开头添加distrobox容器名称。

{% asset_img distrobox.webp distrobox效果 %}

## 上配置！

```toml
palette = "catppuccin_mocha"

[username]
disabled = false
style_user = "bold green"   # 普通用户颜色
style_root = "bold red"     # root 用户颜色
show_always = true          # 始终显示用户名

[status]
disabled = false
format = '[$status]($style) '   # 显示格式，例如 [1]
style = "bold red"              # 失败时的颜色
map_symbol = true                # 使用默认符号映射
pipestatus = true                # 显示管道状态
pipestatus_separator = "|"       # 管道状态分隔符
pipestatus_format = '[$pipestatus]($style) '

[directory]
truncation_length = 0           # 工作路径截断长度，设置为0则不截断（home折叠与此项无关）

[palettes.catppuccin_mocha]
rosewater = "#f5e0dc"
flamingo = "#f2cdcd"
pink = "#f5c2e7"
mauve = "#cba6f7"
red = "#f38ba8"
maroon = "#eba0ac"
peach = "#fab387"
yellow = "#f9e2af"
green = "#a6e3a1"
teal = "#94e2d5"
sky = "#89dceb"
sapphire = "#74c7ec"
blue = "#89b4fa"
lavender = "#b4befe"
text = "#cdd6f4"
subtext1 = "#bac2de"
subtext0 = "#a6adc8"
overlay2 = "#9399b2"
overlay1 = "#7f849c"
overlay0 = "#6c7086"
surface2 = "#585b70"
surface1 = "#45475a"
surface0 = "#313244"
base = "#1e1e2e"
mantle = "#181825"
crust = "#11111b"

```