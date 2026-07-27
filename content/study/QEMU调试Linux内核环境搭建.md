---
title: "QEMU 调试 Linux 内核环境搭建"
slug: "qemu-linux-kernel-debugging"
date: 2026-07-22
description: "用 Linux 内核源码、BusyBox rootfs、QEMU 和 GDB 搭建一个可运行、可断点调试的内核实验环境。"
tags:
  - "Linux"
  - "内核调试"
  - "QEMU"
  - "GDB"
managedBy: publish-openclaw-notes
---

## 元信息

- 类型：技术博客 / 实验环境教程
- 作者/机构：知乎「玩转Linux内核」
- 原文链接：https://zhuanlan.zhihu.com/p/499637419
- 阅读日期：2026-07-22
- 关联主题：QEMU、GDB、Linux 内核调试、BusyBox rootfs
- 阅读状态：粗读并整理为操作笔记

## 一句话结论

这篇文章的核心价值是：搭一个最小但可调试的 Linux 内核运行环境。它用 `bzImage` 作为内核镜像，用 BusyBox 做最小根文件系统，用 QEMU 启动虚拟机，再用 GDB 连接 QEMU，在宿主机上给内核函数打断点。

## 阅读目的

- 后续读 Linux 内核源码时，不只停留在静态阅读，而是能在 QEMU 里跑起来验证调用路径。
- 建立“内核镜像 + rootfs + 启动参数 + GDB 远程调试”的基本实验模型。
- 理解常见 QEMU/GDB 参数，之后调系统调用、VFS、调度、内存管理时能复用。

## 整体流程

```text
下载并编译 Linux 内核
  -> 得到 bzImage 和 vmlinux
编译 BusyBox
  -> 提供 shell 和基础命令
制作 rootfs.img
  -> 作为虚拟机根文件系统
用 QEMU 启动内核
  -> 运行一个最小 Linux 系统
用 GDB 连接 QEMU
  -> 给内核函数打断点调试
```

这里要分清两个关键文件：

| 文件 | 作用 |
| --- | --- |
| `bzImage` | 压缩后的 Linux 内核启动镜像，QEMU 用 `-kernel` 加载它启动系统 |
| `vmlinux` | 未压缩、带符号信息的内核 ELF 文件，GDB 用它解析函数名、源码行和变量 |

## 核心概念

| 概念 | 中文解释 | 在本文中的作用 |
| --- | --- | --- |
| QEMU | 机器模拟器/虚拟化工具，可以模拟一台 x86_64 机器来启动内核 | 提供可控的内核运行环境 |
| BusyBox | 把很多 Unix 常用命令打包成一个小程序 | 用来构造最小 rootfs，提供 shell、`ls`、`mount` 等命令 |
| rootfs | root filesystem，根文件系统 | 内核启动后需要挂载它，才能找到 `/bin/sh`、`/etc`、`/proc` 等 |
| `bzImage` | x86 下常见的压缩内核镜像 | QEMU 启动内核时加载 |
| `vmlinux` | 编译得到的未压缩内核 ELF 文件 | GDB 调试时加载符号 |
| KASLR | Kernel Address Space Layout Randomization，内核地址空间随机化 | 调试时通常关闭，否则静态符号地址和运行地址可能对不上 |
| GDB remote debugging | GDB 远程调试 | GDB 连接 QEMU 暴露的调试端口，对虚拟机内核下断点 |

## 1. 编译 Linux 内核

下载内核源码：

```bash
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/linux-4.14.191.tar.gz
```

命令解释：

| 部分 | 解释 |
| --- | --- |
| `wget` | 从网络下载文件 |
| URL | Linux Kernel Archives 镜像地址，本文用的是 `4.14.191` |

解压源码：

```bash
tar -xvf linux-4.14.191.tar.gz
```

命令解释：

| 参数 | 解释 |
| --- | --- |
| `tar` | 打包/解包工具 |
| `-x` | extract，解包 |
| `-v` | verbose，显示处理过程 |
| `-f` | 指定要处理的归档文件 |

进入源码目录并生成默认配置：

```bash
cd linux-4.14.191
export ARCH=x86
make x86_64_defconfig
make menuconfig
```

命令解释：

| 命令 | 解释 |
| --- | --- |
| `cd linux-4.14.191` | 进入内核源码根目录 |
| `export ARCH=x86` | 指定目标架构为 x86；后续 `make` 会按 x86 架构构建 |
| `make x86_64_defconfig` | 生成 x86_64 默认内核配置 `.config` |
| `make menuconfig` | 打开交互式配置菜单，用来修改内核编译选项 |

调试内核时需要重点改两个配置：

```text
Kernel hacking  --->
    [*] Kernel debugging
    Compile-time checks and compiler options  --->
        [*] Compile the kernel with debug info
        [*] Provide GDB scripts for kernel debugging

Processor type and features  --->
    [ ] Randomize the address of the kernel image (KASLR)
```

解释：

| 配置 | 为什么需要 |
| --- | --- |
| `Kernel debugging` | 打开内核调试相关能力 |
| `Compile the kernel with debug info` | 让 `vmlinux` 包含调试符号，GDB 才能按函数名和源码调试 |
| `Provide GDB scripts for kernel debugging` | 生成/提供辅助 GDB 调试内核的脚本 |
| 关闭 `KASLR` | 避免内核运行地址随机化导致断点地址不稳定 |

开始编译：

```bash
make -j 20
```

命令解释：

| 参数 | 解释 |
| --- | --- |
| `make` | 按内核 Makefile 构建内核 |
| `-j 20` | 并行启动最多 20 个编译任务，加快编译速度；数字应按本机 CPU 核心数调整 |

编译完成后重点产物：

```text
linux-4.14.191/arch/x86_64/boot/bzImage
linux-4.14.191/vmlinux
```

## 2. 配置 BusyBox

BusyBox 用来给最小系统提供用户态工具。下载和解压文章没有给出完整 URL，但流程是：

```bash
tar -jxvf busybox-1.32.0.tar.bz2
cd busybox-1.32.0
make menuconfig
```

命令解释：

| 命令/参数 | 解释 |
| --- | --- |
| `tar -jxvf` | 解压 `.tar.bz2` 格式归档；`-j` 表示通过 bzip2 解压 |
| `cd busybox-1.32.0` | 进入 BusyBox 源码目录 |
| `make menuconfig` | 打开 BusyBox 配置菜单 |

需要打开静态编译：

```text
Settings  --->
    [*] Build BusyBox as a static binary (no shared libs)
```

为什么要静态编译：

- 静态编译会把 BusyBox 需要的库尽量打进二进制里。
- 最小 rootfs 里不需要额外准备复杂的动态链接库。
- 对刚搭调试环境的人来说，启动失败概率更低。

## 3. 制作 rootfs

创建一个 10 MiB 的磁盘镜像文件：

```bash
dd if=/dev/zero of=rootfs.img bs=1M count=10
```

命令解释：

| 参数 | 解释 |
| --- | --- |
| `dd` | 按块复制数据，常用于创建镜像文件 |
| `if=/dev/zero` | input file，从无限输出 0 的设备读取数据 |
| `of=rootfs.img` | output file，写入到 `rootfs.img` |
| `bs=1M` | 每次写 1 MiB |
| `count=10` | 写 10 次，所以得到约 10 MiB 镜像 |

把镜像格式化为 ext4：

```bash
mkfs.ext4 rootfs.img
```

命令解释：

| 部分 | 解释 |
| --- | --- |
| `mkfs.ext4` | 在目标文件或分区上创建 ext4 文件系统 |
| `rootfs.img` | 被格式化成 ext4 的镜像文件 |

挂载镜像，准备写入文件：

```bash
mkdir fs
sudo mount -t ext4 -o loop rootfs.img ./fs
```

命令解释：

| 命令/参数 | 解释 |
| --- | --- |
| `mkdir fs` | 创建挂载点目录 |
| `sudo mount` | 以管理员权限挂载文件系统 |
| `-t ext4` | 指定文件系统类型为 ext4 |
| `-o loop` | 把普通文件当作块设备挂载 |
| `rootfs.img` | 要挂载的镜像文件 |
| `./fs` | 挂载到当前目录下的 `fs` |

把 BusyBox 安装进 rootfs：

```bash
sudo make install CONFIG_PREFIX=./fs
```

命令解释：

| 部分 | 解释 |
| --- | --- |
| `make install` | 执行 BusyBox 的安装目标 |
| `CONFIG_PREFIX=./fs` | 指定安装根目录为 `./fs`，也就是写入刚挂载的 rootfs |

补充最小目录和配置：

```bash
cd fs
sudo mkdir proc dev etc home mnt
sudo cp -r ../examples/bootfloppy/etc/* etc/
sudo chmod -R 777 .
cd ..
```

命令解释：

| 命令 | 解释 |
| --- | --- |
| `mkdir proc dev etc home mnt` | 创建最小 Linux 系统常见目录 |
| `proc` | 后续可挂载 procfs，查看内核导出的进程和系统信息 |
| `dev` | 放设备节点，例如控制台、磁盘设备 |
| `etc` | 放启动脚本和配置 |
| `home` | 用户目录 |
| `mnt` | 临时挂载点，后面挂共享磁盘会用到 |
| `cp -r ../examples/bootfloppy/etc/* etc/` | 拷贝 BusyBox 示例启动配置 |
| `chmod -R 777 .` | 粗暴放开 rootfs 内文件权限；实验环境可用，正式系统不应这样做 |

卸载 rootfs：

```bash
sudo umount fs
```

解释：写入完成后必须卸载，让缓存数据刷回 `rootfs.img`。否则镜像可能内容不完整。

## 4. 用 QEMU 启动内核

基础启动命令：

```bash
qemu-system-x86_64 \
  -kernel ./linux-4.14.191/arch/x86_64/boot/bzImage \
  -hda ./busybox-1.32.0/rootfs.img \
  -append "root=/dev/sda console=ttyS0" \
  -nographic
```

命令解释：

| 参数 | 解释 |
| --- | --- |
| `qemu-system-x86_64` | 启动 x86_64 机器模拟器 |
| `-kernel .../bzImage` | 指定要启动的 Linux 内核镜像 |
| `-hda rootfs.img` | 把 `rootfs.img` 作为第一块 IDE 磁盘，进入虚拟机后通常是 `/dev/sda` |
| `-append "..."` | 传给 Linux 内核的启动参数 |
| `root=/dev/sda` | 告诉内核根文件系统在 `/dev/sda` |
| `console=ttyS0` | 把内核控制台输出到串口 |
| `-nographic` | 不开图形窗口，把串口输入输出放到当前终端 |

退出 QEMU：

```text
Ctrl-a c
```

解释：在 `-nographic` 模式下，先按 `Ctrl-a`，松开后再按 `c`，进入 QEMU monitor，再执行退出命令。不同环境也可用 `Ctrl-a x` 直接退出。

## 5. 用 GDB 调试内核函数

启动 QEMU 时加入调试参数：

```bash
qemu-system-x86_64 \
  -kernel ~/linux-4.14.191/arch/x86_64/boot/bzImage \
  -hda ~/busybox-1.32.0/rootfs.img \
  -append "root=/dev/sda console=ttyS0" \
  -s -S \
  -smp 1 \
  -nographic
```

命令解释：

| 参数 | 解释 |
| --- | --- |
| `-s` | `-gdb tcp::1234` 的简写，让 QEMU 在 1234 端口等待 GDB 连接 |
| `-S` | QEMU 启动后先暂停 CPU，等 GDB 发 `continue` 后再运行 |
| `-smp 1` | 虚拟机只模拟 1 个 CPU，降低调试复杂度 |

在另一个终端启动 GDB：

```bash
gdb ./linux-4.14.191/vmlinux
```

命令解释：

| 部分 | 解释 |
| --- | --- |
| `gdb` | GNU Debugger |
| `vmlinux` | 带符号的内核 ELF 文件，用于解析函数名、变量和源码位置 |

连接 QEMU：

```gdb
target remote localhost:1234
```

解释：

| 部分 | 解释 |
| --- | --- |
| `target remote` | GDB 连接远程调试目标 |
| `localhost:1234` | QEMU 暴露在本机 1234 端口的 GDB stub |

给内核函数下断点：

```gdb
b new_sync_read
continue
```

命令解释：

| 命令 | 解释 |
| --- | --- |
| `b new_sync_read` | 在内核函数 `new_sync_read` 入口处打断点 |
| `continue` | 让暂停的虚拟 CPU 继续运行 |

在 QEMU 里的 Linux shell 执行：

```bash
ls
```

为什么 `ls` 会触发断点：

- `ls` 需要读取目录和文件信息。
- 读文件路径会进入 VFS 和具体文件系统读路径。
- `new_sync_read` 是内核同步读路径中的函数之一，所以可能被触发。

这个例子说明：调内核不是只看源码函数，而是通过用户态命令触发真实内核路径，再用 GDB 停在目标函数上观察调用栈和变量。

## 6. 添加共享磁盘

创建 64 MiB 共享磁盘镜像：

```bash
dd if=/dev/zero of=ext4.img bs=512 count=131072
mkfs.ext4 ext4.img
```

命令解释：

| 参数 | 解释 |
| --- | --- |
| `bs=512` | 每块 512 字节 |
| `count=131072` | 写 131072 块，总大小 512 * 131072 = 67108864 字节，即 64 MiB |
| `mkfs.ext4 ext4.img` | 把镜像格式化成 ext4 文件系统 |

QEMU 启动时增加第二块磁盘：

```bash
qemu-system-x86_64 \
  -kernel ~/linux-4.14.191/arch/x86_64/boot/bzImage \
  -hda ~/busybox-1.32.0/rootfs.img \
  -hdb ~/shadisk/ext4.img \
  -append "root=/dev/sda console=ttyS0" \
  -s \
  -smp 1 \
  -nographic
```

命令解释：

| 参数 | 解释 |
| --- | --- |
| `-hda rootfs.img` | 第一块磁盘，虚拟机里通常是 `/dev/sda`，作为根文件系统 |
| `-hdb ext4.img` | 第二块磁盘，虚拟机里通常是 `/dev/sdb`，作为共享盘 |

在 QEMU 虚拟机里挂载共享盘：

```bash
mount -t ext4 /dev/sdb /mnt
```

命令解释：

| 部分 | 解释 |
| --- | --- |
| `mount -t ext4` | 按 ext4 文件系统挂载 |
| `/dev/sdb` | QEMU 中的第二块磁盘 |
| `/mnt` | 挂载点 |

在宿主机挂载同一个镜像：

```bash
mkdir -p share
sudo mount -t ext4 -o loop ext4.img ./share
```

解释：

- 宿主机把 `ext4.img` 当作普通镜像文件挂载到 `share`。
- QEMU 把同一个 `ext4.img` 当作第二块磁盘。
- 这样可以在宿主机和虚拟机之间交换文件。

注意：同一个 ext4 镜像同时被宿主机和虚拟机读写有一致性风险。实验时最好避免两边同时写，写完一边后卸载，再到另一边挂载使用。

## 关键细节

- `bzImage` 用来启动，`vmlinux` 用来调试；不要拿 `bzImage` 给 GDB 当符号文件。
- 调试内核前关闭 `KASLR`，否则断点地址可能和实际运行地址不一致。
- `-s -S` 是 QEMU 内核调试最常用组合：开 GDB 端口，并在第一条指令前暂停。
- `console=ttyS0 -nographic` 让整个虚拟机在终端里交互，适合服务器和纯命令行环境。
- BusyBox 静态编译可以减少动态库依赖，是做最小 rootfs 的省事方案。
- `root=/dev/sda` 依赖 QEMU 磁盘参数；如果换成 virtio 磁盘，设备名可能变成 `/dev/vda`。

## 常见误解

| 误解 | 更准确的理解 |
| --- | --- |
| QEMU 只能跑完整发行版 | QEMU 也可以直接加载一个内核镜像和一个很小的 rootfs |
| 有 `bzImage` 就能 GDB 调试 | GDB 需要 `vmlinux` 的符号信息，`bzImage` 主要用于启动 |
| 断点打不上就是 GDB 坏了 | 很多时候是没开 debug info、没关 KASLR、符号文件不匹配或内核版本不一致 |
| rootfs 只是一个目录 | 在这里 rootfs 是一个 ext4 镜像，挂载后才像目录一样写入内容 |
| 共享磁盘可以两边随便同时写 | ext4 不是为这种双端同时写设计的，容易产生一致性问题 |

## 和当前主题的关系

这篇不是讲某个内核子系统的理论，而是给 Linux 内核阅读提供实验底座。后续读 VFS、系统调用、文件系统、调度、内存管理时，可以用这个环境：

- 在目标内核函数上下断点。
- 用用户态命令触发内核路径。
- 观察调用栈、参数、结构体字段。
- 把“源码里的静态结构”对应到“运行时的真实路径”。

## 疑问

| 问题 | 为什么重要 | 下一步 |
| --- | --- | --- |
| 文章基于 Linux `4.14.191`，当前更高版本内核的配置项和函数名是否变化？ | 后续如果用新版内核复现实验，命令可能需要调整 | 实操时记录当前内核版本，并核对 `new_sync_read` 是否仍存在 |
| rootfs 中是否还需要创建设备节点，例如 `/dev/console`？ | 某些内核启动失败可能是缺少必要设备节点 | 后续实操时补一版更稳的 rootfs 初始化脚本 |
| 共享文件是否应改用 9p/virtiofs 而不是共享 ext4 镜像？ | 双端挂载 ext4 有一致性风险，9p/virtiofs 更适合共享目录 | 后续补充 QEMU `-virtfs` 或 virtiofs 方案 |

## 可迁移启发

- 读内核源码时，最好同时准备“触发命令”。例如读 VFS，不只是看 `fs/read_write.c`，还要设计 `cat`、`ls`、`dd` 之类的触发动作。
- 内核调试环境要区分“运行文件”和“调试符号文件”：运行靠镜像，调试靠符号。
- 最小实验系统越简单，越容易定位问题。先用 BusyBox + 单 CPU + 关闭 KASLR 跑通，再逐步增加复杂度。

## 来源摘记

- 原文说明最小可运行 Linux 系统需要内核镜像 `bzImage` 和 `rootfs`。
- 原文使用 Linux `4.14.191`、BusyBox `1.32.0`、QEMU、GDB 远程调试完成实验。
- 原文示例通过在 `new_sync_read` 打断点，再在虚拟机中执行 `ls` 来触发内核读路径。
