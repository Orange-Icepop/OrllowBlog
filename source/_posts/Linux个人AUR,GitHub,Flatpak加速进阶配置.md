---
title: 'Linux个人AUR,GitHub,Flatpak加速进阶配置'
categories:
  - 系统
tags:
  - Linux
  - ArchLinux
  - 代理
  - mihomo
  - AUR
  - Flatpak
  - GitHub
abbrlink: e6260061
date: 2026-10-23 00:31:31
---

在使用Linux时，虽然各大国内镜像已经可以覆盖大多数使用场景，但是部分场景仍然依赖不稳定的代理，甚至必须直连。对此，需要一些更加激进的手段。本文将结合个人解决方案，介绍针对AUR、GitHub与Flatpak的加速配置。

## GitHub

{% note info %}
此段是AUR章节中一部分加速手段的前置条件。
{% endnote %}

GitHub的加速是老生常谈的问题了，除了教育网内部有专门的出口节点，其他大陆地区网络直接连接都慢的一批。虽然可以使用机场访问，但是下载大文件和git仓库的时候，不光大部分机场的带宽有限，消耗宝贵的流量配额也很令人心疼。对此，不少人贡献了自己的海外服务器来公益性地帮助国内用户下载，像[GitHub Proxy](https://github.akams.cn/)这样的网站就提供了一个聚合这些节点的站点，方便个人用户使用。在GitHub站点页面中，可以使用[GitHub-Fast油猴脚本](https://github.com/laboratorys/GitHub-Fast)来添加代理站点资源下载的链接，不用手动拼接URL了。

但是话又说回来了，授人以鱼不如授人以渔，与其用着别人的加速节点，不如自己部署一个加速节点，既不抢带宽，又不薅慷慨的人的羊毛。其中，最好的方案是使用Cloudflare Workers。Cloudflare作为路人皆知的互联网大善人，Workers有每天10万次的免费请求额度，可以执行很多简单的TypeScript代码，而且使用的是Cloudflare的全球边缘节点，速度虽然肯定比不上国内CDN，但是相比直连肯定稳定得多，并且带宽大部分情况下能吃满几MB/s。

{% note warning %}
### 关于Cloudflare的服务条款对于代理的限制

Cloudflare在2025年12月3日的[用户协议](https://www.cloudflare.com/terms/)第2.2.1款中新增了(j)条，原话是：

{% blockquote Cloudflare Self-Serve Subscription Agreement https://www.cloudflare.com/terms/ %}

Unless otherwise expressly permitted in writing by Cloudflare, you will not and you have no right to:

(j) use the Services to provide a virtual private network or other similar proxy services.

{% endblockquote %}

翻译：除非经过CF特别允许，否则你无权用CF的服务提供虚拟私有网络（VPN）或其他类似的代理服务。

网上众说纷纭，都说用CF做加速被禁了，但是我认为其实并没有。很有说服力的解释是，CF别有用心地把VPN和代理放在了一起，而VPN的核心是“虚拟”和“私有”，Workers上的API端点是公开的、面向所有用户的，并且也没有“构建专有访问通道”的属性，只是简单地转发了资源请求，没有进行额外加密等试图构建私有网络的行为，可以作为抗辩的理由。否则，CF自己的王牌反代业务直接被纳入了默认禁止的范畴，这很显然不是CF的意思。CF的本意应该是让要玩VPN的人去用WARP，而不是浪费Workers的资源做这种无意义的事情，更有可能是为了防止Workers业务被大手肘击。

至于社区里那些广为流传的封号事件，多数是因为反代没有给登录页面加黑名单，导致被安全公司的扫描器误认为是钓鱼网站导致的。对此，下述的两个项目均做了白名单来预防。为了防止被境外服务器扫描，还同时建议在Cloudflare的`域名-安全性-安全规则`中针对这些域名开启一个过滤器，禁止非中国大陆地区的IP访问。

{% asset_img avoid-non-cn-conn.webp 禁止非中国大陆地区访问的WAF配置 %}

{% endnote %}

我个人使用的是[gh-proxy](https://github.com/hunshcn/gh-proxy)项目，有python和Workers两个版本，前者可以放在海外VPS上运行，后者则可以直接放到Workers上运行。

由于有一个和jsdelivr相关的[Bug](https://github.com/hunshcn/gh-proxy/pull/166)，并且原版gh-proxy太久没维护、暂时没合我这个PR，建议fork[我的仓库](https://github.com/Orange-Icepop/gh-proxy)来部署。

然后，只要在目标需要访问的GitHub https链接前加上你在Workers里绑定的URL作为前缀，就能利用Cloudflare的边缘节点加速GitHub访问了。该项目能够覆盖很多使用https协议传输的资源，包括Release资源、Raw资源和使用https协议的git拉取。

{% note warning %}
此代理项目的git push行为未经过验证，也不建议使用该代理进行git push。
{% endnote %}

## AUR

众所周知，`aur.archlinux.org`这个AUR的核心站点天天被打，并且由于其服务器位于境外的缘故，国内直连很慢甚至根本连不通，大部分源码和资源还托管在GitHub上。这导致国内使用AUR的体验并不好。

{% note info %}
在这里说一句题外话，就我个人知道的AUR投毒事件而言，主要瞄准的是无人维护的孤儿包，也就是npm类型的供应链攻击。对于活跃维护的AUR项目，我认为投毒难度和xz投毒事件不相上下。因此，只要注意一点，关注AUR仓库的上游项目进行交叉验证，就没什么问题。
{% endnote %}

### RPC & PKGBUILD

我使用的AUR助手是使用Rust编写的paru。paru的配置文件`/etc/paru.conf`里有两个很有用的配置：

```conf
[options]
AurUrl = https://aur.archlinux.org
AurRpcUrl = https://aur.archlinux.org/rpc
```

前者是paru下载PKGBUILD等元数据的端点。paru使用`Git Smart HTTP`协议来获取PKGBUILD，因此那些反代`/cgit/aur.git/snapshot/*.tar.gz`的方案是不可使用的。

```
GET  https://aur.archlinux.org/<pkgbase>/info/refs?service=git-upload-pack
POST https://aur.archlinux.org/<pkgbase>/git-upload-pack
```

后者是通过RPC来应答`paru -Ss`等操作的端点。RPC端点可以使用[goaurrpc](https://github.com/moson-mo/goaurrpc)项目来自行实现，但是需要解决的不只是RPC端点，这使得反代方案仍然具有更大的可行性。

我编写了一个[Cloudflare Workers项目](https://github.com/Orange-Icepop/CF-Aur-Acc)来实现反代加速，可以直接部署到Cloudflare上。项目已经做好了白名单限制。然后，根据README里的介绍，把你的AUR助手的配置做出对应修改，就能够加速AUR PKGBUILD下载和RPC请求啦～

### GitHub资源

流量的真正大头是二进制、包和源码的下载。

#### http资源

对于http资源请求，可以通过修改`/etc/makepkg.conf`来指定调用的命令。我编写了一个脚本来将指向GitHub的资源从上一节里讲到的代理站下载：

```sh
#! /bin/bash
# 该脚本用于处理yay/paru安装软件时，由github下载缓慢甚至无法下载的问题
# 检测域名是不是github相关域名，如果是，则在 https://github.com 前添加镜像网站前缀
# 采用wget2代替curl进行8线程下载
{
    proxy_url="<你在上一节里配置/找到的代理站>"
    # 提取协议、域名和路径部分
    protocol=$(echo $2 | cut -f1 -d':')
    domain=$(echo $2 | cut -f3 -d'/')
    path=$(echo $2 | cut -f4- -d'/')

    case "$domain" in
        "github.com"|"gist.github.com"|"raw.githubusercontent.com")
            # 排除 commit 路径
            if [[ $path =~ ^[^/]+/[^/]+/commit/ ]]; then
                url=$2
            else
                # 在原始URL前添加镜像前缀
                url="https://$proxy_url/$2"
                unset http_proxy https_proxy all_proxy
                unset HTTP_PROXY HTTPS_PROXY ALL_PROXY
            fi
            ;;
        *)
            url=$2
            ;;
    esac

    echo "Original URL: $2"
    echo "Downloading from: $url"
    echo "Saving to: $1"
    echo "----------------------------------------"

    # 使用wget2进行多线程下载并显示进度
    /usr/bin/wget2 --max-threads 16 --progress bar --report-speed bytes -O $1 "$url"

    # 检查axel退出状态
    if [ $? -eq 0 ]; then
        echo "Download completed successfully!"
    else
        echo "Download failed!"
        exit 1
    fi
}
```

我原本用的是axel来下载，但是似乎在处理某些资源的时候和代理站八字不合，导致文件大小会有微小出入，通不过资源检查，无奈只能改用[wget2](https://github.com/rockdaboot/wget2)。wget2可以从AUR安装。

然后：

```conf /etc/makepkg.conf
DLAGENTS=('file::/usr/bin/curl -qgC - -o %o %u'
          'ftp::/usr/bin/curl -qgC - -o %o %u'
          'http::/path/to/your/script.sh %o %u'
          'https::/path/to/your/script.sh %o %u'
          'rsync::/usr/bin/rsync --no-motd -z %u %o'
          'scp::/usr/bin/scp -C %u %o')
```

就可以了。

#### git仓库

依然是利用之前的GitHub代理站。通过git的`insteadOf`配置，可以透明地从代理站拉取git仓库。

{% note warning %}
执行以下操作时请注意，`insteadOf`只能覆盖http协议，没有能力也绝对不能覆盖SSH协议（会造成SSH密钥交换失败）。如果你在日常往GitHub上推代码的时候用的是http协议，请将这些需要有上传的仓库设置为使用SSH。否则，请勿执行以下操作。如果你平常只下载`*-bin`的AUR包，上述的脚本已经足够了。
{% endnote %}

执行以下命令：

```bash
git config --global url."https://代理站域名/https://github.com/".insteadOf https://github.com/
```

由于大部分AUR上的`*-git`包都是使用HTTP协议拉取的GitHub仓库，这个命令可以覆盖到大多数git包的仓库拉取。

## Flathub

### 基础配置

国内建议使用[CERNET镜像](https://help.mirrors.cernet.edu.cn/flathub/)。这是一个背景很硬的智能重定向站点，全名为`教育网联合镜像站 MirrorZ`，能够将下载源重定向到离你最近的高校镜像上。

```bash
sudo flatpak remote-modify flathub --url=https://mirrors.cernet.edu.cn/flathub
```

其中，数据量较大的是appstream元数据，单次更新大小约10MB。然而，国内仅有的两个Flathub高校镜像站中，USTC源明确如果未命中appstream缓存，将会重定向到`dl.flathub.org`上。这导致很多情况下，appstream的更新需要花上10分钟以上。

### appstream加速

最简单的方法：强制让`flathub.org`走代理。修改mihomo配置文件，添加：

```yaml
- DOMAIN-SUFFIX,flathub.org,PROXY
```

将`PROXY`换成你配置里面的代理节点。理想状况下，这将使appsteam的更新时间缩短到10秒左右。
