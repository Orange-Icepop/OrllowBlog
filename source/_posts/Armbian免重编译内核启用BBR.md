---
title: Armbian免重编译内核启用BBR
categories:
  - 系统
tags:
  - 嵌入式
  - Linux
  - Debian
  - ARM
abbrlink: 5a24709e
date: 2026-09-19 01:31:22
---

## 问题

Armbian 的部分内核配置里没有 BBR，这对于部分将开发板作为家用服务器的人来说是硬伤。。以 Allwinner sunxi64 的 `current` 分支6.18.33 为例，内核 `.config` 里是：

```
# CONFIG_TCP_CONG_BBR is not set
```

结果是 `net.ipv4.tcp_available_congestion_control` 里看不到 `bbr`，`sysctl`设 `net.ipv4.tcp_congestion_control=bbr` 会失败。

简单而言有三条路：

1. 换内核
2. 用 Armbian 的 build 框架重编整个内核
3. 把 BBR 编成一个树外（out-of-tree）模块。

如果自己重新编译内核，不仅耗时耗力，还有可能丢失部分驱动和兼容性，因此本文选择自行编译树外模块。BBR算法由tcp_bbr.c提供且独立度很高，可以使用modprobe热加载，只要使用对应的内核headers，从对应的内核源码自行编译tcp_bbr.ko，然后丢到`/lib/modeules/{内核名称}`下面，就可以直接modprobe了。

目前主要有两种方法：在设备上原生编译（方案 A），在 x64 主机上用 qemu + Docker 编译（方案 B）。两者产出的 `.ko` 是一样的，ARM设备如果CPU性能足够可以选方案 A ，性能不大够的话就选 B 。

## 确认内核状态

在目标设备上执行：

```bash
uname -r
```

假设输出 `6.18.33-current-sunxi64`。后面所有步骤的版本号都以它为准。

```bash
# 内核配置：有 /proc/config.gz 就用它，否则用 /boot 下的
CONF=/boot/config-$(uname -r)
if [ -r /proc/config.gz ]; then zcat /proc/config.gz > /tmp/kconfig; CONF=/tmp/kconfig; fi
grep -E 'CONFIG_(MODULES|TCP_CONG_BBR|MODVERSIONS|MODULE_SIG|CFI_CLANG|LTO_CLANG)' "$CONF"

which insmod modprobe depmod
```

要确认的几点：

| 配置项 | 期望 | 说明 |
|---|---|---|
| `CONFIG_MODULES` | `=y` | 不支持模块加载的内核，这条路走不通 |
| `CONFIG_TCP_CONG_BBR` | 未设置或 `=m` | 如果是 `=y`，内核里已经有了，不用编 |
| `CONFIG_MODVERSIONS` | 最好未设置 | 开着的话符号 CRC 必须与内核匹配 |
| `CONFIG_MODULE_SIG` | 最好未设置 | 开着的话模块要签名才能加载 |
| `CONFIG_CFI_CLANG` / `CONFIG_LTO_CLANG` | 未设置 | 开着的话模块必须用同款编译器编 |

记下 `uname -r` 的完整输出和 `CONFIG_MODVERSIONS` 的状态，验证环节要用到。

## 关于树外模块的原理

树外模块把链接推迟到 `insmod`：编译阶段只用头文件，未定义的符号由内核在加载时用运行中内核的符号表解析。下面每条都直接影响成败。

- **headers 必须与运行内核版本一致。** 这是最容易出问题的一环。版本不一致时，轻则编译报结构体成员不存在，重则编译通过但加载报 `Invalid module format` 或 `Unknown symbol`。
- **编译不需要内核源码，但 headers 包必须是一棵完整可用的构建树**（含生成的头文件、`scripts/`、`Module.symvers`、`.config`）。只拷一个 `include/` 目录编不动。
- **`.c` 只能用内核已导出的符号。** `tcp_bbr.c` 满足这一点（只引用 9 个导出符号），所以单个文件就能编成模块——这是树外方案成立的前提。
- **`.ko` 才是要用的文件**，`.o` 是中间产物。一次 `make` 会全部产出，拷到设备上的是 `.ko`。
- **运行时打补丁由内核自动完成**（alternatives、ftrace 等），不需要任何额外配置。你在构建阶段什么都不用做。
- **编译器用哪个通常无所谓。** vermagic 里不含编译器信息，kbuild 顶多打印一条版本不一致的 warning。例外见上面表格里的 CFI/LTO。
- **BTF 相关的提示可以忽略。** 构建时出现 `Skipping BTF generation`、加载时出现 `missing module BTF` 都属正常，6.18 里这条只是告警，不影响 BBR 注册。

## 方案 A：在设备上原生编译

最省事的方案。要求设备上能装 headers 包、有 `make` 和 `gcc`。

### A1. 装 headers

```bash
sudo apt update
sudo apt install -y linux-headers-current-sunxi64
```

包名由内核 flavour 决定，用这条确认：

```bash
uname -r                                  # 6.18.33-current-sunxi64
apt-cache search linux-headers | grep sunxi64
```

装完检查构建目录链接：

```bash
ls -l /lib/modules/$(uname -r)/build
# 应指向 /usr/src/linux-headers-6.18.33-current-sunxi64

ls /usr/src/linux-headers-$(uname -r)/Module.symvers
```

第二组命令必须有输出。Armbian 的 headers 包安装时会在本机编译 `scripts/` 下的工具，等几分钟属正常。

如果 apt 源里只有比运行内核更新的版本（例如仓库已推进到 6.18.44，而设备还在 6.18.33），不要直接装，否则 headers 与运行内核不匹配。用 B1 的方法取指定版本的 `.deb` 再装。

### A2. 取一份对应版本的 tcp_bbr.c

`tcp_bbr.c` 不在 headers 包里，从对应版本的内核源码取：

```bash
VER=6.18.33
MAJOR=${VER%%.*}
cd /tmp
curl -fLO "https://cdn.kernel.org/pub/linux/kernel/v${MAJOR}.x/linux-${VER}.tar.xz"
tar -xf "linux-${VER}.tar.xz" "linux-${VER}/net/ipv4/tcp_bbr.c"
```

版本必须与 `uname -r` 的版本号部分一致。取出来的是 `/tmp/linux-6.18.33/net/ipv4/tcp_bbr.c`。

上游文件通常可直接用。若编译时报某个结构体成员不存在，说明该发行版对 `net/ipv4/` 打过补丁，需要改用发行版源码包里的同名文件。

### A3. 写 Makefile

新建工作目录，把 `tcp_bbr.c` 放进去，同目录写 `Makefile`：

```makefile
obj-m := tcp_bbr.o
KDIR ?= /lib/modules/$(shell uname -r)/build
PWD  := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

`all` 和 `clean` 下面的命令行**必须是 TAB 开头**，用空格会报 `missing separator`。

在设备上 `uname -r` 返回的就是目标内核，所以这里可以直接用。

### A4. 编译

```bash
make
```

产物：

```
tcp_bbr.o       编译 tcp_bbr.c 得到，中间产物
tcp_bbr.mod.c   自动生成
tcp_bbr.mod.o
tcp_bbr.ko      要用的文件
```

### A5. 验证

```bash
modinfo -F vermagic tcp_bbr.ko
uname -r
```

两条的第一个字段必须一致。后面的 `SMP preempt mod_unload` 等标志来自 headers 的 `.config`，用对了 headers 就会匹配。

如果内核 `CONFIG_MODVERSIONS=y`，再比对一下符号版本：

```bash
modprobe --dump-modversions tcp_bbr.ko | grep tcp_register_congestion_control
grep tcp_register_congestion_control /lib/modules/$(uname -r)/build/Module.symvers
```

两边的 CRC 要相同。

### A6. 加载

```bash
sudo install -m 0644 tcp_bbr.ko /lib/modules/$(uname -r)/extra/
sudo depmod -a
sudo modprobe tcp_bbr
```

然后启用：

```bash
cat /proc/sys/net/ipv4/tcp_available_congestion_control
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
cat /proc/sys/net/ipv4/tcp_congestion_control
```

BBR 需要 pacing，配 `fq` 队列规则（`eth0` 换成实际出口网卡）：

```bash
sudo tc qdisc replace dev eth0 root fq
```

开机自动加载：

```bash
echo tcp_bbr | sudo tee /etc/modules-load.d/bbr.conf
echo 'net.ipv4.tcp_congestion_control = bbr' | sudo tee /etc/sysctl.d/99-bbr.conf
```

## 方案 B：在 x64 主机上用 qemu + Docker

适合设备上没有编译环境，或者不想在设备上装一堆 `-dev` 包的情况。

这里不是传统意义的交叉编译，而是在 x64 上跑一个 **arm64 的 Docker 容器**，容器里是目标的 aarch64 原生工具链，由 qemu-user + binfmt_misc 执行。好处是工具链与设备上的发行版一致，省掉配交叉工具链的麻烦；代价是编译速度受模拟影响，比方案 A 慢。

### B1. 取对应版本的 headers .deb

不要用仓库当前索引里的版本，它可能比设备内核新。Armbian 的 pool 里保留历史版本，按文件名里的内核版本挑：

```bash
VER=6.18.33
BASE=https://mirrors.tuna.tsinghua.edu.cn/armbian/pool/main/l/linux-headers-current-sunxi64
DEB=$(curl -sL "$BASE/" | grep -oE '[^"/]+\.deb' | grep -F "__${VER}-" | sort -u | tail -1)
echo "$DEB"
curl -fLO "$BASE/$DEB"
```

文件名形如 `linux-headers-current-sunxi64_26.5.1_arm64__6.18.33-S8365-...deb`。包名是 `linux-headers-current-sunxi64`，内核版本只体现在文件名和 control 的 `Provides` 字段里：

```bash
dpkg-deb -f "$DEB" Package Version Provides
# Provides: linux-headers (= 6.18.33), ...
```

设备上如果能装，`apt-get download linux-headers-current-sunxi64` 更省事，但要先确认拉到的版本与 `uname -r` 一致。

### B2. 注册 binfmt

这一步决定 arm64 容器能不能跑起来：

```bash
docker run --rm --privileged tonistiigi/binfmt --install arm64
```

验证：

```bash
docker run --rm --platform linux/arm64 debian:trixie uname -m
# 期望输出 aarch64
```

**宿主机每次重启后都要重做这一步**，binfmt_misc 的注册不持久。忘了的话所有 arm64 容器都会报 `exec format error`。

### B3. 写 Dockerfile

新建空目录，放入 Dockerfile，并把 B1 下到的 `.deb` 放在同一目录：

```dockerfile
FROM --platform=linux/arm64 debian:trixie

# trixie 用 deb822 格式的源文件，位于 /etc/apt/sources.list.d/debian.sources。
# 用 http 是因为基础镜像里没有 ca-certificates，走 https 会报
# certificate verify failed。apt 本身仍通过 Debian 的 archive keyring 校验包。
RUN sed -i 's|http://deb.debian.org|http://mirrors.tuna.tsinghua.edu.cn|g' \
        /etc/apt/sources.list.d/debian.sources \
 && apt-get update \
 && apt-get install -y --no-install-recommends \
        make gcc libc6-dev bison flex libssl-dev libelf-dev pahole dwarves \
        bc kmod cpio xz-utils ca-certificates \
 && rm -rf /var/lib/apt/lists/*

COPY *.deb /tmp/headers.deb
RUN dpkg -i /tmp/headers.deb && rm -f /tmp/headers.deb
```

`linux-headers-current-sunxi64` 的依赖是 `make gcc libc6-dev bison flex libssl-dev libelf-dev pahole`，上面覆盖了，其余是它安装时编译 `scripts/` 和日常使用需要的。

`COPY *.deb` 要求目录里只有一个 `.deb`，有多个就改成完整文件名。

### B4. 构建镜像

```bash
docker build --platform linux/arm64 -t bbr-build .
```

安装 headers 阶段会编译 `fixdep`、`modpost`、`resolve_btfids`，全在 qemu 模拟下跑，需要几分钟。

### B5. 写 Makefile 并编译

这一步和方案 A 有一处关键差别。在 arm64 容器里：

| 命令 | 返回 |
|---|---|
| `uname -m` | `aarch64` |
| `uname -r` | **宿主机内核版本**，例如 `6.12.107+deb13-amd64` |
| `gcc -dumpmachine` | `aarch64-linux-gnu` |

容器不是虚拟机，它共用宿主机的内核，所以 `uname -r` 拿到的是宿主机版本。`KDIR` 因此**不能**写成 `$(shell uname -r)`，必须写死目标内核：

```makefile
obj-m := tcp_bbr.o

# 容器里 uname -r 是宿主机内核，必须写死
KDIR ?= /lib/modules/6.18.33-current-sunxi64/build
PWD  := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=arm64 CROSS_COMPILE= modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=arm64 CROSS_COMPILE= clean
```

命令行同样必须 TAB 开头。`CROSS_COMPILE=` 留空，因为容器里本来就是 aarch64 原生工具链。`tcp_bbr.c` 的获取方式同 A2。

把 `tcp_bbr.c` 和 `Makefile` 放进一个模块目录，用容器编译：

```bash
MODDIR=$PWD/module
mkdir -p "$MODDIR"
# 放入 tcp_bbr.c 和 Makefile

docker run --rm --platform linux/arm64 \
    -v "$MODDIR:/work/module" -w /work/module \
    bbr-build make -j"$(nproc)"
```

确认产物是 arm64：

```bash
file "$MODDIR/tcp_bbr.ko"
# ELF 64-bit LSB relocatable, ARM aarch64, ...
```

### B6. 容器里不要做的事

- **不要 `insmod` / `modprobe`。** 容器共用宿主机内核，往 x64 宿主机加载一个 arm64 模块只会失败。
- **不要指望在这里做加载测试。** 宿主机是 x86_64，加载不了 aarch64 模块。能做的验证是 vermagic 匹配和符号可解析，见下一节。

### B7. 一个环境相关的坑

如果宿主机的 `$HOME` 不可写（某些沙箱环境如此），`docker build` 会因为无法创建 `$HOME/.docker` 而失败：

```
ERROR: mkdir /opt/dsh/data/.docker: read-only file system
```

把配置目录指到可写位置即可：

```bash
export DOCKER_CONFIG=/path/to/writable/.docker
mkdir -p "$DOCKER_CONFIG"
```

`docker run` 和 `docker pull` 通常不受影响，只有 `build` 会撞上，所以现象看起来像 BuildKit 的问题，实际不是。

## 验证

拿到 `.ko` 之后，在设备上跑这三条：

```bash
# 1. 架构
file tcp_bbr.ko

# 2. vermagic 必须与运行内核一致
modinfo -F vermagic tcp_bbr.ko
uname -r

# 3. 依赖的符号是否都由内核导出
for s in $(nm -u tcp_bbr.ko | awk '{print $2}'); do
    awk -F'\t' -v s="$s" '$2==s{n++} END{printf "%-36s %d\n", s, n+0}' \
        /lib/modules/$(uname -r)/build/Module.symvers
done
```

第 3 条每一项都是 0 才说明全部可解析。

`Module.symvers` 是制表符分隔的三列 `<crc> <symbol> <module>`，所以上面用 `awk` 按字段比对。不要用 `grep "\t$s\t"`：基本正则里 `\t` 不代表制表符，会全部匹配不到，看起来像是符号都没导出。

## 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| `insmod: ... Invalid module format` | vermagic 不匹配 | `dmesg \| tail` 会打印 `version magic ... should be ...`，对照两边的内核版本和 flavour |
| `insmod: ... Unknown symbol in module` | 内核没有导出该符号 | 换用与运行内核完全同版本的 headers 和 `tcp_bbr.c` |
| `insmod: ... Required key not available` | 内核开了 `CONFIG_MODULE_SIG` 且模块未签名 | 只能用签名后的模块，或换内核 |
| 编译报 `missing separator` | Makefile 命令行用了空格 | 改成 TAB |
| `unable to find /lib/modules/.../build` | headers 没装或版本不对 | `ls -l /lib/modules/$(uname -r)/build` |
| 编译报结构体成员不存在 | `tcp_bbr.c` 版本与内核不匹配 | 换成对应版本的文件 |
| 容器里 `exec format error` | binfmt 未注册或重启后丢失 | 重跑 B2 |
| 容器里找不到 `KDIR` 路径 | Makefile 用了 `$(shell uname -r)` | 写死目标内核路径 |
| `modprobe: FATAL: Module tcp_bbr not found` | 未 `depmod`，或 `.ko` 不在 `/lib/modules/$(uname -r)/` 下 | 拷进 `extra/` 后 `sudo depmod -a` |
| 加载成功但 `sysctl` 认不出 bbr | 模块没真正加载 | `lsmod \| grep tcp_bbr`，看 `dmesg` |
| dmesg 出现 `missing module BTF, cannot register kfunc` | 树外构建不生成 BTF，属预期 | 忽略，不影响 BBR 注册 |
| 构建日志出现 `Skipping BTF generation ...` | headers 包不含 vmlinux，属预期 | 忽略 |

最后两条补充一句：老一些的内核里，缺少模块 BTF 会让注册函数返回错误、导致模块加载失败（而不是只告警）。6.18 改成了只告警。如果你编的是别的版本且加载失败，先去 `dmesg` 里确认有没有这条，再怀疑别的地方。

## 补充：在 x64 上做真正的交叉编译

不需要 qemu，也不需要 Docker，速度最快，适合反复迭代。

```bash
sudo apt install -y gcc-aarch64-linux-gnu make bison flex \
    libssl-dev libelf-dev pahole
```

headers 的 `.deb` 是 arm64 架构，不能 `dpkg -i` 到 x64 主机上，改成解包：

```bash
dpkg-deb -x linux-headers-current-sunxi64_*.deb ./sysroot
KDIR=./sysroot/usr/src/linux-headers-6.18.33-current-sunxi64
```

解包出来的树里，`scripts/` 下的工具还没编（正常安装时由 postinst 完成）。手动补上：

```bash
make -C "$KDIR" ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
make -C "$KDIR" ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j"$(nproc)" scripts
make -C "$KDIR" ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j"$(nproc)" M=scripts/mod
```

这三步用的是宿主 gcc，编出来的是 x64 可执行文件，这正是需要的。然后编模块：

```bash
make -C /path/to/module \
     ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
     KDIR="$KDIR" modules
```

Makefile 里把 `KDIR` 参数化：

```makefile
obj-m := tcp_bbr.o
KDIR ?= /lib/modules/6.18.33-current-sunxi64/build
PWD  := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules
```

如果目标内核开了 `CONFIG_CFI_CLANG` 或 `CONFIG_LTO_CLANG`，这条路走不通，必须用 Clang。

## 附录：一次实测记录

给 Armbian sunxi64 的 `6.18.33-current-sunxi64` 编译 `tcp_bbr`，源码取自上游 `linux-6.18.33/net/ipv4/tcp_bbr.c`（未修改），用方案 B。

目标内核配置：

```
# CONFIG_TCP_CONG_BBR is not set
# CONFIG_MODVERSIONS is not set
# CONFIG_MODULE_SIG is not set
CONFIG_MODULES=y
CONFIG_CC_IS_GCC=y
CONFIG_CC_VERSION_TEXT="aarch64-linux-gnu-gcc (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0"
CONFIG_LTO_NONE=y
CONFIG_RANDSTRUCT_NONE=y
```

headers 包：

```
Package: linux-headers-current-sunxi64
Version: 26.5.1
Provides: linux-headers (= 6.18.33)
Description: Armbian Linux current headers 6.18.33-current-sunxi64
```

产物：

```
tcp_bbr.o    600664 bytes   ELF 64-bit LSB relocatable, ARM aarch64, with debug_info
tcp_bbr.ko   745120 bytes   ELF 64-bit LSB relocatable, ARM aarch64, with debug_info
vermagic: 6.18.33-current-sunxi64 SMP preempt mod_unload aarch64
```

9 个依赖符号在 `Module.symvers` 中全部命中：

```
__warn_printk  alt_cb_patch_nops  get_random_u8  jiffies  memset
minmax_running_max  register_btf_kfunc_id_set
tcp_register_congestion_control  tcp_unregister_congestion_control
```

构建日志里唯一一条警告是编译器版本差异：内核由 GCC 13.3.0（Ubuntu）构建，模块用了 GCC 14.2.0（Debian）。因为 `CONFIG_MODVERSIONS` 未设置，不影响加载。
