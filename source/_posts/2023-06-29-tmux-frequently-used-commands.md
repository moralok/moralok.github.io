---
title: Tmux 常用命令、快捷键和个人实践
date: 2023-06-29 18:44:44
tags: [tmux]
---

本文记录了 `Tmux` 常用命令、快捷键和个人实践作为备忘笔记。

> Tmux 发音 "tee-muhks" (tˈiː.mʌks).

<!-- more -->

## Tmux 是什么

`Tmux` 是一个终端复用器（`Terminal Multiplexer`），它可以将终端的会话和窗口解绑分离，达到“窗口关闭，会话不终止，并且还能使用窗口再次绑定目标会话”的效果，从而避免因为网络中断或者窗口关闭导致会话中运行的进程终止。

### 安装

`Ubuntu server 20.04` 及后续版本默认已安装 `Tmux`。如果没有，可以通过以下命令安装：

```bash
sudo apt update
sudo apt-get install tmux
```

更多安装方式参考官方 Wiki 文档 [Installing tmux](https://github.com/tmux/tmux/wiki/Installing)。

## 基本用法

### 会话（Sesssion）管理

**命令**

| 操作 | 命令 | 说明 |
|---|---|---|
| 新建会话 | `tmux new -s <session-name>` | |
| 分离会话 | `tmux detach` | |
| 查看会话列表 | `tmux ls` | |
| 接入会话 | `tmux attach -t <session-name>` | 按名称接入 |
| 杀死会话 | `tmux kill-session -t <session-name>` | 按名称指定会话 |
| 切换会话 | `tmux switch -t <session-name>` | 按名称切换 |
| 重命名会话 | `tmux rename-session -t <old> <new>` | 指定旧名称和新名称 |
| 重命名当前会话 | `tmux rename-session <new>` | |

**快捷键**

| 操作 | 快捷键 | 说明 |
|---|---|---|
| 分离当前会话 | `Ctrl+b d` | `detach` |
| 查看并选择会话 | `Ctrl+b s` | `session` |
| 重命名当前会话 | `Ctrl+b $` | |
| 发送 EOF | `Ctrl+d` | 在空的 shell 提示符下通常会退出当前 shell |

`Ctrl+d` 不是 tmux 的前缀快捷键，而是向当前窗格中的程序发送 EOF。若 shell 因此退出，tmux 会关闭对应窗格；如果这是窗口里的最后一个窗格，窗口也会关闭；如果该窗口是会话中的最后一个窗口，会话就会结束。若前台运行的是其他程序，`Ctrl+d` 的效果取决于该程序如何处理输入结束。


### 窗口（Window）管理

**命令**

| 操作 | 命令 | 说明 |
|---|---|---|
| 新建窗口 | `tmux new-window -n <window-name>` | 指定窗口名称 |
| 关闭窗口 | `tmux kill-window -t <window-name>` | 按名称关闭 |
| 关闭当前窗口 | `tmux kill-window` | |
| 切换窗口 | `tmux select-window -t <window-name>` | 按名称切换 |
| 切换窗口 | `tmux select-window -t <window-id>` | 按 ID 切换 |
| 重命名窗口 | `tmux rename-window -t <old> <new>` | 指定旧名称和新名称 |
| 重命名当前窗口 | `tmux rename-window <new>` | |
| 交换当前窗口 | `tmux swap-window -t <target-id>` | 与指定编号的窗口交换位置 |
| 交换指定窗口 | `tmux swap-window -s <source-id> -t <target-id>` | 指定源窗口和目标窗口 |
| 移动窗口 | `tmux move-window -t <target-id>` | 将当前窗口移动到指定位置 |

**快捷键**

| 操作 | 快捷键 | 说明 |
|---|---|---|
| 新建窗口 | `Ctrl+b c` | 状态栏会显示窗口信息 |
| 切换到上一个窗口 | `Ctrl+b p` | `prev` |
| 切换到下一个窗口 | `Ctrl+b n` | `next` |
| 切换到指定窗口 | `Ctrl+b <window-id>` | 按 ID 切换 |
| 查看并选择窗口 | `Ctrl+b w` | `window` |
| 重命名当前窗口 | `Ctrl+b ,` | |
| 关闭当前窗口 | `Ctrl+b &` | |


### 窗格（Pane）管理

**命令**

| 操作 | 命令 | 说明 |
|---|---|---|
| 划分窗格 | `tmux split-window [-v]` | `vertical` |
| 划分窗格 | `tmux split-window -h` | `horizontal` |
| 移动焦点 | `tmux select-pane -U` | 向上，`upper` |
| 移动焦点 | `tmux select-pane -D` | 向下，`down` |
| 移动焦点 | `tmux select-pane -L` | 向左，`left` |
| 移动焦点 | `tmux select-pane -R` | 向右，`right` |
| 交换窗格 | `tmux swap-pane -U` | 与上一个窗格交换位置 |
| 交换窗格 | `tmux swap-pane -D` | 与下一个窗格交换位置 |

**快捷键**

| 操作 | 快捷键 | 说明 |
|---|---|---|
| 上下划分窗格 | `Ctrl+b "` | |
| 左右划分窗格 | `Ctrl+b %` | |
| 切换窗格 | `Ctrl+b <arrow key>` | 使用方向键选择窗格 |
| 切换到上一个窗格 | `Ctrl+b ;` | |
| 切换到下一个窗格 | `Ctrl+b o` | |
| 与上一个窗格交换位置 | `Ctrl+b {` | |
| 与下一个窗格交换位置 | `Ctrl+b }` | |
| 关闭当前窗格 | `Ctrl+b x` | |
| 拆分为独立窗口 | `Ctrl+b !` | |
| 全屏显示当前窗格 | `Ctrl+b z` | 再按一次恢复原大小；状态栏会显示大写的 Z |
| 显示窗格编号 | `Ctrl+b q` | |


### 其他命令

**命令**

| 操作 | 命令 | 说明 |
|---|---|---|
| 列出所有快捷键及对应的 tmux 命令 | `tmux list-keys` | |
| 列出所有 tmux 命令及其参数 | `tmux list-commands` | |

**快捷键**

| 操作 | 快捷键 | 说明 |
|---|---|---|
| 进入翻页模式 | `Ctrl+b [` | |
| 退出翻页模式 | `q` | 在翻页模式中按下 |
| 打开 tmux 命令提示符 | `Ctrl+b :` | 在 tmux 内直接输入管理命令，按 `Enter` 执行 |
| 取消命令提示符 | `Esc` | |

> 场景举例：在实时查看日志时，想要交换窗口顺序（默认没有绑定快捷键），可以打开 tmux 命令提示符执行 `swap-window` 命令而无需中断当前进程。


## 个人实践

### 会话管理

1. 使用命名会话：方便查找和管理
2. 使用分离和重连：保持会话连续性
3. 避免嵌套：不要在 `tmux` 会话中通过 `SSH` 连接另一个 `tmux` 会话

### 基础配置

使用 `~/.tmux.conf` 管理配置。

1. 开启鼠标支持：在配置文件中加入 `set -g mouse on`，支持用鼠标点击切换窗格、拖拽调整大小及滚轮查看历史输出。
2. 版本控制配置文件：将你的 `~/.tmux.conf` 存入 `Git` 仓库进行多机同步。参考 [dotfiles](https://github.com/moralok/dotfiles)。

> 个人没有修改默认前缀键 `Ctrl+b` 为可能更顺手的 `Ctrl+a` 或 `Ctrl+s`；没有在配置文件中加入 `setw -g mode-keys vi` 开启 `vi` 模式并绑定系统剪贴板，**现代终端通常支持选中即复制（Copy on select）**。

### 窗格与窗口操作

1. 善用临时最大化：当某个窗格日志过多时，使用 `Ctrl+b z` 临时全屏放大当前窗格，再次按下即可还原。
2. 避免过度分屏：使用窗口并行管理不同任务。
3. 结合 `AI Agent` 工作流：使用固定的分屏比例，如左侧大窗格运行 `Agent`，右侧窗格查看日志或执行命令。

### 自动化与脚本化

1. 一键生成工作环境：针对固定开发项目，编写 `Shell` 脚本利用 `tmux new-session`、`split-window` 和 `send-keys` 自动创建指定数量的窗口、分屏并自动运行初始化命令。
2. 使用和管理插件：通过 `TPM`（`Tmux Plugin Manager`）使用和管理插件
    - [安装 TPM](https://github.com/tmux-plugins/tpm)
    - 推荐安装 [tmux-sensible](https://github.com/tmux-plugins/tmux-sensible)，统一的基础开箱即用设置，修改不合理的默认配置
    - 推荐安装 [tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect)，手动保存与恢复会话，不惧重启和断电
    - 推荐安装 [tmux-continuum](https://github.com/tmux-plugins/tmux-continuum)，自动备份会话，配合 tmux-resurrect 每 15 分钟后台静默保存

如果服务器发生重启，默认的 `tmux` 会话会全部丢失。在安装 `tmux-resurrect` 后：
- 手动保存快照：在 `tmux` 会话中按下快捷键 `Ctrl+b`，然后按 `Ctrl+s`。
- 手动恢复快照：在 `tmux` 会话中按下快捷键 `Ctrl+b`，然后按 `Ctrl+r`。

如果还安装了 `tmux-continuum`，无需手动恢复。

### 配置文件（仅供参考）

```conf
# Mouse support
set -g mouse on

# Window and pane numbering
set -g base-index 1
setw -g pane-base-index 1
set -g renumber-windows on

# Terminal
set -g default-terminal "tmux-256color"
set -sg escape-time 10

# Reload configuration
bind R source-file ~/.tmux.conf \; display-message "tmux config reloaded"

# Plugins
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'

# Continuum: automatic save every 15 minutes
# automatic restore (on startup)
set -g @continuum-restore 'on'

# Initialize TMUX plugin manager (keep this line at the very bottom of tmux.conf)
run '~/.tmux/plugins/tpm/tpm'
```


## 补充说明

### 会话名称和 ID

我之前误以为，（不给定名称）新建会话时系统显示的数字就是会话 ID。其实每个会话都有名称和 ID 两个独立字段；`tmux ls` 默认显示的是名称。可以用下面的命令同时查看两者：

```bash
tmux list-sessions -F '#{session_name} #{session_id}'
```

输出中第一列是会话名称，第二列是会话 ID：

```text
5 $5
blog $1
dev $2
docker $4
```

第一行的名称 `5` 和 ID `$5` 只是数字恰好相同；它们仍是不同字段。

## 参考链接

- [Tmux 使用教程](https://www.ruanyifeng.com/blog/2019/10/tmux.html)
- [Tmux 学习](https://blog.csdn.net/weixin_51311218/article/details/122172339)
- `AI`
