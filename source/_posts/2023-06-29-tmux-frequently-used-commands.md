---
title: Tmux 常用命令和快捷键
date: 2023-06-29 18:44:44
tags: [tmux]
---

本文记录了 `Tmux` 常用命令和快捷键以及参考 `AI` 提供的最佳实践形成的个人实践作为备忘笔记。

<!-- more -->

## Tmux 是什么

`Tmux` 是一个终端复用器（`Terminal Multiplexer`），它可以将终端的会话和窗口解绑分离，达到窗口关闭，会话不终止，并且还能使用窗口再次绑定目标会话的效果，从而避免因为网络中断或者窗口关闭导致会话中运行的进程终止。

`Ubuntu server 20.04` 及后续版本默认已安装 `Tmux`。如果没有，可以通过以下命令安装：

```bash
sudo apt update
sudo apt-get install tmux
```

## 基本用法

### 会话管理

#### 新建会话

```bash
tmux new -s <session-name>
```

#### 分离会话

```bash
tmux detach
```

#### 查看会话列表

```bash
tmux ls
```

#### 接入会话

```bash
tmux attach -t <session-name>
tmux attach -t <session-id>
```

#### 杀死会话

```bash
tmux kill-session -t <session-name>
tmux kill-session -t <session-id>
```

#### 切换会话

```bash
tmux switch -t <session-name>
tmux switch -t <session-id>
```

#### 重命名会话

```bash
tmux rename-session -t <old> <new>
tmux rename-session <new>
```

#### 会话快捷键

> 辅助记忆：detach，session

- **`Ctrl+b d`：分离当前会话**
- **`Ctrl+b s`：查看会话列表并选择**
- **`Ctrl+b $`：重命名当前会话名称**
- **`Ctrl+d`：关闭当前会话**


### 窗口管理

#### 新建窗口

```bash
tmux new-window -n <window-name>
```

#### 关闭窗口

```bash
tmux kill-window -t <window-name>
tmux kill-window
```

#### 切换窗口

```bash
tmux select-window -t <window-name>
tmux select-window -t <window-id>
```

#### 重命名窗口

```bash
tmux rename-window -t <old> <new>
tmux rename-window <new>
```

#### 窗口快捷键

- **`Ctrl+b c`：创建一个新窗口，状态栏会显示窗口信息**
- `Ctrl+b p`：切换到上一个窗口，prev
- `Ctrl+b n`：切换到下一个窗口，next
- `Ctrl+b <number>`：切换到指定编号的窗口
- **`Ctrl+b w`：查看窗口列表并选择，window**
- **`Ctrl+b ,`：重命名当前窗口**
- **`Ctrl+b &`：关闭当前窗口**


### 窗格管理

#### 划分窗格

> 辅助记忆：vertical，horizontal

```bash
tmux split-window [-v]
tmux split-window -h
```

#### 移动光标

> 辅助记忆：upper，down，left，right。

```bash
tmux select-pane -U
tmux select-pane -D
tmux select-pane -L
tmux select-pane -R
```

#### 交换窗格

当前窗格和上一个或下一个窗格交换位置。

```bash
tmux swap-pane -U
tmux swap-pane -D
```

#### 窗格快捷键

- **`Ctrl+b "`：上下划分窗格**
- **`Ctrl+b %`：左右划分窗格**
- **`Ctrl+b <arrow key>`：使用方向键切换窗格**
- `Ctrl+b ;`：光标切换到上一个窗格
- `Ctrl+b o`：光标切换到下一个窗格
- `Ctrl+b {`：当前窗格和上一个窗格交换位置
- `Ctrl+b }`：当前窗格和下一个窗格交换位置
- **`Ctrl+b x`：关闭当前窗格**
- `Ctrl+b !`：将当前窗格拆分为一个独立窗口
- `Ctrl+b z`：当前窗格全屏显示，再使用一次会变回原来大小，状态栏会有大写的 Z 标识
- `Ctrl+b q`：显示窗格编号


### 其他命令

#### 进入翻页模式

- **`Ctrl+b [`：进入翻页模式**
- `q`：退出翻页模式

#### 列出所有快捷键，及其对应的 Tmux 命令

```bash
tmux list-keys
```

#### 列出所有 Tmux 命令及其参数

```bash
tmux list-commands
```


## 个人实践

### 会话管理

1. 使用命名会话：方便查找和管理
2. 使用分离和重连：保持会话连续性
3. 避免嵌套：在 `tmux` 会话中通过 `SSH` 连接另一个 `tmux` 会话

### 基础配置

使用 `~/.tmux.conf` 管理配置。

1. 开启鼠标支持：在配置文件中加入 `set -g mouse on`，支持用鼠标点击切换窗格、拖拽调整大小及滚轮查看历史输出。
2. 版本控制配置文件：将你的 `~/.tmux.conf` 存入 `Git` 仓库进行多机同步。

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

如果服务器发生重启，默认的 `tmux` 会话会全部丢失。

在安装 `tmux-resurrect` 后：
- 手动保存快照：在 `tmux` 会话中按下快捷键 `Ctrl+b`，然后按 `Ctrl+s`。
- 手动恢复快照：在 `tmux` 会话中按下快捷键 `Ctrl+b`，然后按 `Ctrl+r`。

如果还安装了 `tmux-continuum`，无需手动恢复。

### 配置文件（仅供参考）。

```shell
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

# Continuum: automatic save and restore
set -g @continuum-save-interval '15'
set -g @continuum-restore 'on'

# Initialize TMUX plugin manager (keep this line at the very bottom of tmux.conf)
run '~/.tmux/plugins/tpm/tpm'
```


## 参考链接

- [Tmux 使用教程](https://www.ruanyifeng.com/blog/2019/10/tmux.html)
- [Tmux 学习](https://blog.csdn.net/weixin_51311218/article/details/122172339)
- `AI`