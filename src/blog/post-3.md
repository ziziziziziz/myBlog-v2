---
title: 在虚拟机上安装 Arch Linux（三）
date: 2026-02-04
description: 磁盘分区、格式化、挂载与系统安装。
---

本篇文章尚不完善，对其导致的阅读理解障碍作者表示抱歉，并将会尽快补充完善此文章...

## 磁盘分区

确认磁盘有 `lsblk` 和 `fdisk -l` 两种命令。
![检查磁盘](@images/post-3/检查磁盘.png)

### 分区类型介绍

分区类型有三种：
- **EFI系统分区（ESP）**
- **Linux Filesystem 分区（根分区）**
- **swap 交换区**

关于 EFI 分区的大小，<a href="https://wiki.archlinuxcn.org/wiki/EFI_系统分区" target="_blank"  rel="noopener noreferrer">官方文档</a>有详细表述。
![EFI分区大小](@images/post-3/EFI分区大小.png)

### 分区工具选择

分区主要有两种方式：`cfdisk` 或 `fdisk`。根据 DeepSeek 的分析，最终选定创建分区表 GPT 的方式为 `cfdisk`；不创建 swap 分区，以后若有需要用 swap 文件代替，以减小当前安装时的操作复杂度。

**DeepSeek 的分析**：

- `fdisk`：纯命令行式，功能更底层、强大，支持所有高级操作，但学习成本较高（对新手来说如果找到合适的指南，按照指南输入字母其实不难，作者安装第一个 Arch Linux 时使用的分区工具就是 `fdisk`）。
- `cfdisk`：文本图形界面（TUI），对当前分区状态、操作对象的展示一目了然，极大降低误操作风险。覆盖了 90% 的常用分区操作（创建、删除、修改类型、调整大小），对于标准安装（GPT，分 1-3 个区），`cfdisk` 功能完全足够且更高效。

---

### 实际操作

```bash
cfdisk /dev/sda   # 启动分区工具
```

后面具体的操作过程忘记了，这里引用极客技术博客（[Arch Linux VirtualBox 安装教程](https://geek-blogs.com/blog/arch-linux-vbox/)）的教程：

1. 打开后选择 **gpt** 分区表类型，回车确认。
2. 然后新建分区，点击「New」，输入大小 **512M**，类型选择 **EFI System**。
3. 剩余空间新建根分区：大小默认（剩余全部），类型 **Linux filesystem**。
4. （可选）若需 swap，在根分区前预留空间，类型选择 **Linux swap**。
5. 点击「Write」保存分区表，输入 `yes` 确认，然后「Quit」退出。

![dev-sda1](@images/post-3/dev-sda1.png)
![dev-sda2](@images/post-3/dev-sda2.png)

---

## 一个小插曲

按下 `Ctrl + C` 键后命令提示符变成了 `130 root@archiso ~ #`，并且右侧出现了 `:(` 符号。问 DeepSeek，它说查不到关于 `:(` 符号的参考资料，但应该没有什么影响。
![live环境下的空CtrlC](@images/post-3/live环境下的空CtrlC.png)

---

## 文件系统选择

文件系统有多种类型：Btrfs、ext4、FAT32 等。选择根目录文件系统为 **ext4**。

**DeepSeek 的分析**：

- **FAT32**：对于根目录来说已经过时，主要用于 EFI 分区，是 EFI 分区的标准文件系统格式。
- **ext4**：稳定、可靠、高性能、广泛兼容、学习难度低，历经 Linux 系统 20 年考验。
- **Btrfs**：集高级功能于一身，学习难度高。可瞬间创建系统快照，并在子卷级别进行灵活管理；自动检测数据损坏，并配合冗余（如 RAID1）进行修复；透明压缩（如 zstd, lzo），可节省空间并提升 I/O 性能。
- 如果你的首要目标是“成功安装并体验 Arch Linux 系统”，那么选择 **ext4**。

此外，简明指南中提到：“若使用 Swap 文件而不是 Swap 分区，在 Btrfs 文件系统中设置休眠（hibernate）的时候需要额外的步骤，而且可能有兼容性问题。”
我的主要目标是学习 Arch 系统，不想把太多精力花在别的事情上，所以选择了 ext4 作为根目录文件系统。

### 格式化分区

```bash
mkfs.fat -F32 /dev/sda1   # 格式化 EFI 分区（FAT32）
mkfs.ext4 /dev/sda2       # 格式化根分区（ext4）
```

---

## 推荐资源

推荐一本书：<a href="https://www.linuxprobe.com/" target="_blank"  rel="noopener noreferrer">《Linux就该这么学》</a>

---

## 挂载分区

### 什么是挂载？

基于作者的理解，Arch 系统被成功安装之前，在一切的开始，也就是现在，系统的 `/dev/sda1`（ESP）和 `/dev/sda2`（根分区）这两个分区是独立的。要安装 Arch 需要让它们联合起来发挥作用，并且需要先找一个地方（`/mnt`）来放置它们才能操作。

“放置”不是把根分区搬到 `/mnt` 里，而是让 `/mnt` 成为根分区，这就是挂载。说得严谨一点就是把 `/mnt` 链接到 `/dev/sda2`。

在 `/mnt` 成为根目录后，就可以把 ESP（EFI系统分区） 装到里面（挂载到 `/mnt/boot` 里）。

### ESP 挂载路径说明

在网络上的很多旧教程里，ESP 是被挂载到 `/mnt/boot/efi` 里的。DeepSeek 是这样说的：

- 挂载到 `/mnt/boot/efi`：历史遗留方案。GRUB 等引导程序曾默认使用此路径。
- 挂载到 `/mnt/boot`：**新手首选，追求最简单安装流程**（这也是官方推荐）。
- 挂载到 `/mnt/efi`：配置稍复杂；需确保引导程序能找到内核。追求系统整洁、多系统共存的进阶用户选用。

### 执行挂载

```bash
mount /dev/sda2 /mnt                      # 挂载根分区到 /mnt
mount --mkdir /dev/sda1 /mnt/boot         # 挂载 ESP 到 /mnt/boot
df -h                                    # 使用 df 命令检查挂载情况
```

这里的 `mount --mkdir /dev/sda1 /mnt/boot` 命令参照的是官方给出的格式（很多教程里没有这样写）。其中的 `--mkdir` 参数的官方解释是：

> Allow to make a target directory (mountpoint) if it does not exist yet.

即如果挂载点不存在就先创建挂载点，这样可以省略使用命令 `mkdir -p /mnt/boot` 创建 `/boot` 目录这一步。

> **注意**：在挂载之后，对 `/mnt` 里文件的一切操作相当于是对被安装系统的文件的操作，将会被保留到安装后的新系统里。

---

## 安装系统

### 安装基础包

```bash
pacstrap /mnt base base-devel linux linux-firmware vim
```

各种教程在这里的命令不尽相同，但至少都包含 `base linux linux-firmware`。出于谨慎，作者搜了一下各命令的不同之处。

#### `-K` 参数

官网说：“`-K` 标志会初始化一个新的 pacman 密钥环，而不是使用主机的密钥环。对于在 chroot 环境中使用非官方镜像并希望从主机导入其密钥的人来说，可以省略此标志。”

DeepSeek 的解释：

> - **不加 `-K`**：新系统会复制 Live 安装环境（主机）里现有的所有软件包签名密钥。这就像从旧房子（Live 环境）把所有钥匙串（密钥环）直接带到新房子（新系统），能立即打开所有门（验证软件包）。这是最常用、最省事的做法。
> - **加上 `-K`**：新系统会丢弃 Live 环境的所有旧钥匙，当场打造一套全新的、完全空白的钥匙串。这能保证绝对干净，但新系统里一把钥匙都没有，后续需要你自己手动添加所有必要的钥匙（密钥）。
> - 你使用的是受信任的镜像站，已经严格校验了 ISO 的 PGP 签名，确保了安装源的纯净，这样 Live 环境的密钥本身就是官方的、干净的。把这个干净的密钥环复制过去，新系统就能立刻信任并安装所有官方软件包。这是推荐做法。使用 `-K` 会产生额外步骤，你需要在新系统里手动配置 pacman 密钥环，这对于初学者来说既复杂又没必要。

#### `base-devel` 包

软件开发工具链，包含编译器（`gcc`）、`make`、`autoconf`、`sudo` 等，在后续的安装过程中是必须用到的。

小插曲：在官网搜索这个包时，看到了 Edge 浏览器的有趣翻译
![Edge高级翻译](@images/post-3/Edge的高级翻译.png)

#### `vim` 包

作者希望在新系统里使用 vim 编辑器。

### 安装前确认

在执行命令前可以先用以下命令打开 mirrorlist 文件确认一下是否有把国内镜像源提前：

```bash
vim /etc/pacman.d/mirrorlist
```

输入 `:q` 并回车退出 vim 编辑器。（这里的 vim 是 live 环境自带的，新安装的 Arch 系统里没有 vim）

### 安装后的警告

安装后，我在输出文本的末尾几行看到了错误和警告。
![pacstrap警告](@images/post-3/pacstrap警告.png)
询问 DeepSeek 得知所有警告源于虚拟控制台键盘映射与字体的配置文件 `/etc/vconsole.conf` 不存在，DeepSeek 说这是正常的，继续下一步即可。

我半信半疑，在网上搜索后发现有人遇到了同样的问题（<a href="https://forum.archlinuxcn.org/t/topic/15201" target="_blank"  rel="noopener noreferrer">Arch Linux 中文论坛相关讨论</a>），问题时间在 2025 年 11 月，距离现在竟然还不到 4 个月。看到有网友说不影响使用才放心。（这个论坛相对活跃，论坛里有很多好心的大佬帮助解决问题，作者在此注册了账号）

---

## 生成 fstab 文件

生成 fstab 文件以使需要的文件系统（如启动目录 `/boot`）在启动时被自动挂载：

```bash
genfstab -U /mnt > /mnt/etc/fstab   # 生成 fstab 文件
cat /mnt/etc/fstab                  # 检查生成的 /mnt/etc/fstab 文件是否正确
```

命令解释：`cat` 用于将指定文件里的内容输出到终端

---

## Chroot 到新系统

chroot 到新安装的系统，即进入新安装的系统内部，这样就可以直接对系统进行操作了：

```bash
arch-chroot /mnt
```

---

## 基础系统配置

```bash
# 设置时区
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# 生成 /etc/adjtime 并将系统时间写入硬件时钟（假设硬件时钟为 UTC）
hwclock --systohc

# 启用网络时间同步，防止未来时钟漂移
timedatectl set-ntp true
```