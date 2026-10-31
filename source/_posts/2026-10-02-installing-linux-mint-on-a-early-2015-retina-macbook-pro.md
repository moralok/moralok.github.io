---
title: 在 Early 2015 Retina MacbookPro 上安装 Linux Mint
date: 2026-10-02 03:55:01
tags: [linux, mint, mac]
---

这台 `MacBookPro12,1` 是 2015 年款的 13 英寸 Retina 屏 MacBook Pro，配置为 i5-5257U、16GB 内存和一块 256GB 的 `APPLE SSD SM0256G`，系统停留在 `macOS Catalina 10.15.7`。购买于 2017 年初，大概有将近四年没被认真使用了：不少新版本的软件无法安装，使用时容易发热卡顿。

一直以来没动它，原因比较简单——担心有些资料没拷出来，或者旧环境哪天要用。

最近和多位朋友聊到硬件的价格话题，又经常刷到 Mac mini M6 的开箱视频，不免想到这台机器。看着它的二手平均价格仅大约550元-750元，而 Time Machine 占用的 PSSD T7 大约1000元，越想越膈应。10 月 1 日心血来潮，下决心抹盘安装 Linux。

很幸运没踩大坑很快就安装完了，整体顺序如下：

1. 先**体检**，确认这台机器的硬件还健康，值得折腾；
2. 再备份，把可能有价值的文件再次整理备份；
3. 然后制作启动盘、进 Live 环境**验证硬件**；
4. 最后擦掉 macOS 安装 Linux Mint。

> 整个过程是相对谨慎地边查资料边和 ChatGPT 对话确认每一步的。

<!-- more -->

## 体检：先确认它不会装完不久就坏

最怕的不是装不上，而是装完没几天机器废了。所以在 macOS 上先用系统自带的工具把关键部件检查一遍。

### 电池

```console
$ system_profiler SPPowerDataType
      Charge Remaining (mAh): 4793
      Full Charge Capacity (mAh): 5346
      Cycle Count: 510
      Condition: Service Recommended
```

510 次循环不算多，满充容量 5346 mAh，按设计容量算大概还有八成。系统评估为 `Service Recommended` 说明电池确实老了，但这台机器准备长期插电用于 Homelab——2015 年的机器再花一笔钱换电池也不划算，所以这一项不构成阻碍。

### SSD

```console
$ diskutil info disk0
   Device / Media Name:       APPLE SSD SM0256G
   Protocol:                  PCI
   SMART Status:              Verified
```

系统评估为 `SMART Status: Verified`，说明硬盘没有故障。

文件系统另外检查一遍：

```console
$ diskutil verifyVolume disk1
Checking snapshot 1 of 2 (com.apple.TimeMachine.2023-07-20-232507.local)
error: Cross Check : Mismatch between extentref entry reference count (1)
and calculated fsroot entry reference count (0) for extent (0x1c890c2 + 1)
Checking snapshot 2 of 2 (com.apple.TimeMachine.2026-09-28-170833.local)
...
Verifying allocated space
The volume /dev/disk0s2 appears to be OK
Storage system check exit code is 0
```

检查结果 `exit code 0`，正常。中间有一条 `error`，是 2023 年一个本地 Time Machine 快照的引用计数对不上。因为要抹盘重装，所以不用 `diskutil repairVolume` 修复。

### 内存、主板和散热

关机，开机按住 `D`，进入 `Apple Diagnostics`，选择语言后自动开始检测。结果：

```text
No issues found
```

内存是焊在主板上的，无法更换，所以这一项特别要紧。

散热检查：

```console
$ pmset -g therm
Note: No thermal warning level has been recorded
Note: No performance warning level has been recorded
	CPU_Scheduler_Limit 	= 100
	CPU_Available_CPUs 	= 4
	CPU_Speed_Limit 	= 100
```

没有历史热保护记录，CPU 也没有被降频，100 代表满血。

### 两项没能检查出来的

`system_profiler SPNVMeDataType` 没有任何输出。这台机器的 SSD 是 Apple 定制的 PCIe 盘，在 Catalina 下拿不到写入量、剩余寿命这类细节。想要确认可以安装第三方工具。

`diskutil apfs list` 查看空间：

```text
Size (Capacity Ceiling):      250790436864 B (250.8 GB)
Capacity In Use By Volumes:   192571699200 B (192.6 GB) (76.8% used)
Capacity Not Allocated:       58218737664 B (58.2 GB) (23.2% free)
```

用了 76.8%，剩 58.2GB，这个结果可以用于评估“要不要采用双系统”。

> 对我个人而言这一项缺乏参考价值，所有软件和文件都被我视为可以清理。

### 体检小结

| 部件 | 结果 | 判断 |
|---|---|---|
| 电池 | 510 次循环 / 5346 mAh / Service Recommended | 可用，长期插电 |
| SSD | SMART Verified | 无故障迹象 |
| APFS | exit code 0，有一个快照引用计数 error | 可用，未修复 |
| 内存 / 主板 | Apple Diagnostics: No issues found | 正常 |
| 散热 / 降频 | 无 thermal warning，CPU 100% | 正常 |

## 为什么不做双系统

ChatGPT 最初提供的方案里是有双系统的：留 80~100GB 给 macOS，剩下的给 Linux。还有更保守的做法——把 Linux 装在外置 SSD 上。

两个我都没选，甚至可以说我几乎没有认真考虑过它们。一个很重要的原因是我不想双系统的另一半是一个过于陈旧的 macOS，或者是 `OCLP` 强制升级得到的有风险的 macOS。

这台机器闲置了四年，安装 Linux 的重要目的之一就是让它作为 24 小时开机的服务器重新被利用起来。即使最后失败，也可以通过内嵌的“在线恢复模式”重新安装 macOS 挽救。

> 使用一个月以后再回头看，抛开双系统可能存在的其他风险，就硬盘空间而言是完全够用的，容纳一个临时备用的 macOS 好像也不错。

## 为什么是 Mint Cinnamon

真正进入候选的其实就 3 个：长期使用、相对熟悉的 Ubuntu、仅在云服务器上偶尔使用的 Debian、一眼相中名字的 Mint。

我既想要尝点鲜，又不想太过“冒险”，不谈能不能习惯，万一硬件或驱动有问题就很麻烦，我希望它稳定。桌面性能开销不能太大，但也不必为了节省内存过度牺牲桌面体验。

最后版本选的是 Linux Mint 22.3 Cinnamon（代号 Zena），基于 Ubuntu 24.04，内核 6.14。

这一步没有花太多时间纠结，事后看也没错——装完之后除了触控板手势，几乎没有什么需要回头折腾的地方。

## 备份资料

这台机器上的资料在多年前早已备份过一次，所以这次只是上了第二道保险。

拷完以后随机打开几个文件验证：私有 `jar` 能使用，配置文件能读，旧项目代码能打开。

> **检查比备份本身更重要，如果复制了一堆打不开的文件等于没备份。**

## 启动盘：Windows 上的 Rufus

启动盘是在 Windows 上做的。Mac 上也能做（`dd` 或者 `balenaEtcher`），但 `Rufus` 是图形化界面，出错的空间更小。

镜像下载 Linux Mint 22.3 Cinnamon 的 ISO。Rufus 里的设置：

| 选项 | 取值 |
|---|---|
| 设备 | U 盘 |
| 引导类型选择 | Linux Mint 22.3 Cinnamon ISO |
| 分区类型 | GPT |
| 目标系统 | UEFI（非 CSM） |
| 卷标 | 默认 |
| 文件系统 | FAT32（默认） |
| 簇大小 | 16K（默认） |

写入过程中 `Rufus` 提示过一次“另一程序正在使用此驱动器，仍然要格式化它吗”。这是 Windows 那边有进程占着 U 盘，`Rufus` 要先解除占用再重新格式化。确认选中的确实是那根 U 盘，不是电脑硬盘，然后确认。

> 写模式选 `ISO Image mode`，写完状态变成就绪后点“关闭”——不是再点一次“开始”，那会重写一遍。

## Live 环境验证硬件

U 盘插进 MacbookPro，关机，开机时立刻按住 `Option`，启动盘选择界面里会出现 `EFI Boot`，进去之后是 GRUB：

```text
*Start Linux Mint 22.3 Cinnamon 64-bit
 Start Linux Mint 22.3 Cinnamon 64-bit (compatibility mode)
 OEM install (for manufacturers)
 Boot from next volume
 UEFI Firmware Settings
 Memory test
```

第一项就是 Live 模式——系统从 U 盘运行，不修改内部硬盘。`compatibility mode` 是普通模式无法使用时的备选，其余选项跟这次无关。

在 Live 桌面里把该测试的都测试一遍，这是最后一次不付出代价的检查机会：

- Wi-Fi：Broadcom BCM43602 直接就能连，不用装任何驱动
- 触控板：移动、单击、双指滚动、双指右键都正常；双指缩放没有，三指四指手势也没有；双指滚动有点跳，偏快
- 键盘：每一个按键都有反应
- 声音、屏幕（2560×1600）、蓝牙、睡眠唤醒：都正常
- USB：没额外 U 盘，用外置硬盘试的，能识别
- 温度：`sensors` 读到 44°C

Broadcom 无线网卡是这台机器装 Linux 最大的不确定项，网上不少 2015 款 MacBook 的驱动在新内核上编译不过。它在这里直接走 `brcmfmac` 起来了，这一条过掉之后，剩下的都算小事。

> 触控板那两个问题当时判断可以先不管：Live 环境从 U 盘跑，I/O 慢，滚动卡顿可能有一部分是环境造成的；捏合缩放和手势在桌面环境层面本来就不一定有。

## 确认安装：整盘擦除

安装程序里的语言选择 English。

好处是命令输出、日志、报错、文档基本都是英文，遇到问题直接拿报错去搜，界面标签和路径能对上。
另一个顾虑是中文环境会把家目录变成“桌面”“文档”“下载”，平时 `cd`、`find`、写脚本都要多处理一层路径问题。
中文输入法可以装完系统之后再单独加，不影响使用。

安装类型选 `Erase disk and install Linux Mint`，没有手动分区。

> 这里有一个容易出错的地方：从 U 盘启动时，安装程序同时能看到内部 SSD 和 U 盘。引导器（boot loader）必须装到内部 SSD 上，确认目标磁盘是那个约 256GB 的设备再继续。

分区交给安装程序自己处理，结果是：

```console
$ lsblk -f
NAME   FSTYPE FSVER LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda
├─sda1 vfat   FAT32        EDEA-851C                             504.8M     1% /boot/efi
└─sda2 ext4   1.0          90764f1c-d436-4901-9069-e5701f0f9181  207.5G     4% /
```

一个 505MB 的 EFI 分区加一个 207.5GB 的根分区，另外自动建了一个 2GiB 的 `/swapfile`。16GB 内存的机器，swap 这样够用，没必要再调整。

## 意外：满屏 SQUASHFS 报错

在安装完之后没有先点 `Restart Now`，而是先把 U 盘拔了，然后重启。结果是满屏刷报错：

```text
SQUASHFS error: Unable to read fragment cache entry [637a742a]
SQUASHFS error: Unable to read page, block 637a742a, size 8154
I/O error, dev loop0, sector 2299792 op 0x0: (READ) flags 0x800
SQUASHFS error: Failed to read block 0x462ecf32: -5
```

当时第一反应是内部 SSD 出了问题——刚装完就坏，这是最坏的情况。

经分析，`squashfs` 是 Live 镜像内部压缩文件系统，`loop0` 是 U 盘上的镜像文件挂出来的虚拟块设备，这两个东西都在 U 盘这一侧。报错说明系统还在读 U 盘，而 U 盘已经被拔掉，读不到了。

处理方案：长按电源键强制关机，确认 U 盘已经拔掉，正常开机。Mint 从内部 SSD 正常启动，登录进去看 `lsblk -f` 的输出正常，说明安装本身是成功的。

## 新系统基线

第一次启动后先没有装任何软件，只用 `inxi -Fxxxz` 把系统状态记下来。节选：

```text
System:
  Kernel: 6.14.0-37-generic arch: x86_64
  Desktop: Cinnamon v: 6.6.9 tk: GTK v: 3.24.41 wm: Muffin v: 6.6.3
    dm: LightDM v: 1.30.0 Distro: Linux Mint 22.3 Zena base: Ubuntu 24.04 noble
Machine:
  Type: Laptop System: Apple product: MacBookPro12,1 v: 1.0
    UEFI: Apple v: 428.60.3.0.0 date: 10/27/2021
Battery:
  ID-1: BAT0 charge: 36.9 Wh (62.4%) condition: 59.1/74.8 Wh (79.1%) cycles: 511
CPU:
  Info: dual core model: Intel Core i5-5257U
Graphics:
  Device-1: Intel Iris Graphics 6100 vendor: Apple driver: i915
  Monitor-1: eDP-1 model: Apple Color LCD res: 2560x1600 hz: 60
Audio:
  Device-1: Intel Broadwell-U Audio driver: snd_hda_intel
  Server-1: PipeWire v: 1.0.5 status: active
Network:
  Device-1: Broadcom BCM43602 802.11ac Wireless LAN SoC driver: brcmfmac
Drives:
  ID-1: /dev/sda vendor: Apple model: SSD SM0256G size: 233.76 GiB scheme: GPT
Sensors:
  System Temperatures: cpu: 53.0 C pch: 53.5 C
Info:
  Memory: total: 16 GiB available: 15.52 GiB
```

几个值得注意的点：

- 电池在 Linux 下的读数是 59.1/74.8 Wh，79.1%，511 次循环，跟 macOS 报的八成对得上。
- 无线网卡走 `brcmfmac`，蓝牙走 `btusb`，声卡 `snd_hda_intel` 加 PipeWire，显卡 `i915`——这些是老 Mac 装 Linux 最容易出问题的地方，一个都没用额外驱动。
- CPU 53°C。比 Live 环境的 44°C 高，但两次的运行时间、负载、后台任务都不一样，不能直接比较；这个温度对 Broadwell 的双核来说正常。
- 内存 15.52 GiB 可用，缺的那部分是核显占用，16GB 没有跑满。

### 触控板

`xinput list` 里能看到一块 `bcm5974`，这是 MacBook 触控板在 Linux 下的专用输入驱动，由 libinput 接管：

```console
$ xinput list-props 9
	libinput Natural Scrolling Enabled (306):	1
	libinput Tapping Enabled (335):	1
	libinput Tapping Drag Enabled (337):	1
	libinput Disable While Typing Enabled (343):	1
	libinput Scroll Method Enabled (309):	1, 0, 0
	libinput Scrolling Pixel Distance (332):	15
	libinput High Resolution Wheel Scroll Enabled (334):	1
```

自然滚动、轻点即点、轻点拖动、打字时禁用触控板都是开着的；`Scroll Method Enabled: 1, 0, 0` 对应双指滚动启用、边缘滚动和按键滚动关闭；滚动步长是 15 像素。

所以 Live 里感觉到的“跳”不是因为误用了边缘滚动或者第三方驱动，就是这个 15px 的步进配合高分辨率滚轮。Cinnamon 里没有捏合缩放的开关，`gsettings list-recursively org.cinnamon.desktop.peripherals.touchpad` 列出的键里也没有对应项——捏合通常由具体应用自己处理，不是系统级手势。

> 两条终端常用操作：复制是 `Ctrl+Shift+C`，粘贴是 `Ctrl+Shift+V`（因为 `Ctrl+C` 在终端里是中断命令）；另外 Linux 支持“选中即复制”，鼠标选中之后用中键就能粘贴。

## 软件：只装当前要用的

当前定位：这台机器不是 WSL 的复制品，也不是第二台主力开发机，而是一台闲置四年、重新启用的独立 Linux 机器。

第一批只考虑安装了 Git、VS Code 和 Mihomo。

```bash
sudo apt update
sudo apt install git
git config --global user.name "..."
git config --global user.email "..."
```

Mint 的默认源里没有 VS Code，要先加微软官方源，之后 `apt upgrade` 也能跟着一起更新。这比下载 `.tar.gz` 自己解压、自己管 PATH、自己管更新干净得多，卸载也只要 `apt remove code`。

```bash
sudo apt update
sudo apt install wget gpg

wget -qO- https://packages.microsoft.com/keys/microsoft.asc \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/packages.microsoft.gpg > /dev/null

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" \
  | sudo tee /etc/apt/sources.list.d/vscode.list > /dev/null

sudo apt update
sudo apt install code
```

## 现在这台机器在干什么

安装 `Docker` 之后，我花了几天时间，把 Homelab 里原本跑在虚拟机上的几个服务陆续搬了过来，又接上 GitHub Self-hosted Runners 做自动部署。顺带装了 `tmux`、`starship`、`btop` 和几个可有可无的小工具。

大多数时候我是 `ssh` 连接使用，很少打开屏幕登录桌面。所以那些没配好的地方——三指四指手势、捏合缩放、双指滚动那点“跳”——就一直没去调。甚至中文输入法都没有安装。用得少，也就不觉得是个问题。

回头看，有两件事印象比较深。一件是桌面 Linux 的安装越来越省心：`Rufus` 写完盘，Live 环境里试一圈再点几下，分区、引导、驱动基本上都由安装程序和内核自己处理好了。另一件是在这种步骤多、风险分散、但每一步都能验证的事情上，AI 确实好用。我敢说动手就动手的底气，大半来自它把整件事拆成了可以验证的步骤：哪些信息要查、哪些资料要备份、哪一步不可逆、万一失手怎么兜底。抹盘看起来是唯一的单向操作，但 macOS 的在线恢复模式还留着后路，真装砸了也能把系统恢复回来。想清楚这一层，那个按钮按下去就没那么难了。何况这台机器本身也比想象中健康——从 2017 年初购买到现在九年多，只有电池被标成 `Service Recommended`，其余核心部件都还在正常范围内。

> 润色得有一股不烧心短剧味哈哈哈。

它现在是一台 24 小时开着、随时能 `ssh` 进去、跑着几个 Homelab 服务的 Linux 机器。这大概是这台 2015 年的 MacBook 这几年最好的状态。
