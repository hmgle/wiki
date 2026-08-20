---
title: "Linux 按键体系 09：复用层 tmux"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第九篇。
> tmux 在责任链的第 ⑥ 层，位置特殊：**它既是终端里的程序，又是内层程序的终端**。
> 按键要在这里被解码、判断、再编码。

## prefix 的选择

```tmux
set -g prefix M-b
set -g prefix2 C-g # for macOS
unbind C-b
bind M-b send-prefix
```

三个决定，每个都有代价。

### 为什么不用默认的 `C-b`

`Ctrl+b` 在两个高频场景里已经有含义：

- **readline**：`Ctrl+b` = 光标左移一个字符。
- **Vim**：`Ctrl+b` = 向上翻页。

被 tmux 抢走后，这两个操作都要改用别的方式。对 Vim 用户来说 `Ctrl+b` 翻页是常用
操作，代价不小。

### 为什么是 `M-b`

`Alt+b` 在 readline 里是"向左移动一个词"。**这同样有代价，只是更小**——因为在
zsh 里我更常用 `Ctrl+w`（删词）和 fzf 历史搜索，`Alt+b` 的实际使用频率低于
`Ctrl+b`。

而且代价可以挽回：

```tmux
bind M-b send-prefix
```

按两次 `Alt+b`，第二次会把一个字面的 `Alt+b` 发给内层程序。所以 readline 的
"向左移词"仍然可用，只是要多按一次。**"多按一次"和"完全不可用"是两个量级的差别。**

这个技巧对任何 prefix 都适用，是选 prefix 时应该一并配好的。

### `prefix2` 与跨平台

```tmux
set -g prefix2 C-g # for macOS
```

tmux 支持两个 prefix 同时生效。第二个 `Ctrl+g` 是给 macOS 用的——那边的 Option 键
常被用来输入特殊字符，`Alt+b` 不一定送得出来。

`Ctrl+g` 在 readline/emacs 里是"取消当前操作"，代价可接受。

**更重要的是它在 ssh 场景下的价值**：[第六篇](06-who-gets-the-key.html) 讲过 tmux
套 tmux 的问题——本地 tmux 会先吃掉 prefix。有两个不同的 prefix 时，可以让内外层各用
一个，彻底避免歧义。

## 那些真正决定体验的"隐形"设置

比键位绑定更重要的是这几条：

```tmux
# Set the time in milliseconds for which tmux waits after an escape is input
# to determine if it is part of a function or meta key sequences. The default
# is 500 milliseconds.
set-option -s escape-time 10

set -g default-terminal "tmux-256color"
set -s terminal-features[100] "xterm*:RGB"
set -g extended-keys on
set -g extended-keys-format csi-u
```

### `escape-time 10`

[第三篇](03-terminal-keys.html) 讲过：`Alt+x` 编码成 `ESC x`，跟"先按 Esc 再按 x"
字节完全相同，只能靠时间区分。tmux 默认等 **500 毫秒**。

后果是：**在 tmux 里从 Vim 插入模式按 Esc 退出，要等半秒**。

半秒有多长？长到你会不确定"我到底退出了没有"，然后再按一次 Esc。这就是系列主线里
"状态确认成本"最典型的例子——**它消耗的不是时间，是你对系统状态的确信**。

调到 10ms 基本消除这个问题。代价在 [第三篇](03-terminal-keys.html) 说过：高延迟
ssh 下 `ESC` 和后续字节可能被拆进不同的包，间隔超过 10ms，`Alt+x` 就会被误判成
两次独立按键。这是真实的取舍，不是"越小越好"。

### `default-terminal` 与 `terminal-features`

```tmux
set -g default-terminal "tmux-256color"
set -s terminal-features[100] "xterm*:RGB"
```

`default-terminal` 是 tmux **告诉内层程序**的 `TERM` 值；`terminal-features` 是 tmux
**对外层终端**能力的认知。两个方向，别搞混。

`tmux-256color` 而不是 `screen-256color`，因为前者的 terminfo 里包含了斜体、
真彩色等现代能力。`xterm*:RGB` 声明外层终端支持 24 位色。

**常见故障**：ssh 到远端后颜色不对或方向键出乱码，八成是远端没有 `tmux-256color`
这个 terminfo 条目。诊断和修复见 [第六篇](06-who-gets-the-key.html)。

### `extended-keys`

```tmux
set -g extended-keys on
set -g extended-keys-format csi-u
```

CSI u 链条上的中间一环。WezTerm 编码 → **tmux 必须能解码并重新编码** → Neovim 解码。
少了这两行，`Ctrl+Shift+*` 这类组合在 tmux 里就被丢弃了。

需要 tmux 3.2+。

## 键位表

### pane 导航

```tmux
bind k selectp -U # switch to panel Up
bind j selectp -D # switch to panel Down
bind h selectp -L # switch to panel Left
bind l selectp -R # switch to panel Right
```

`hjkl`，和 [第七篇](07-awesome.html) 的 AwesomeWM、[第十一篇](11-neovim.html) 的
Neovim 完全一致。

**跨层一致性在这里的价值最直接**：切窗口、切 pane、切编辑器分屏，是三个不同层的
操作，但方向语义相同。你的手指知道"往左"是 `h`，不需要先想"我现在在哪一层"。

### 窗口与会话

| 键 | 动作 |
| --- | --- |
| `prefix c` | 新建 window（继承当前路径） |
| `prefix "` | 上下分屏（继承当前路径） |
| `prefix %` | 左右分屏（继承当前路径） |
| `prefix L` | 上一个 window |
| `prefix N` | 新建 session |
| `prefix S` | `choose-tree -s` 选 session |
| `prefix M` | 用 fzf 把当前 pane 移到别的 window |
| `prefix t` | 在当前路径开一个浮动 popup |
| `prefix r` | 重载配置 |

三个分屏/新建都加了 `-c "#{pane_current_path}"`：

```tmux
bind '"' split-window -c "#{pane_current_path}"
bind % split-window -h -c "#{pane_current_path}"
bind c new-window -c "#{pane_current_path}"
```

**新 pane 继承当前目录。** 这是个极小的改动，但它消除了一个高频动作："开个新 pane
→ `cd` 到刚才那个目录"。tmux 默认是回到 home 目录，几乎从不是你想要的。

按系列标准评价：它省的不是那次 `cd` 的按键，而是**"我现在在哪个目录"这个问题
根本不需要被问出来**。

`prefix t` 的浮动 popup 也值得一提：

```tmux
# tmux version >= 3.2
bind t display-popup -d "#{pane_current_path}" -xC -yC -w 80% -h 75% -E
```

需要临时跑个命令（`git log`、`man`、算个数）时，popup 覆盖在当前布局上方，关掉后
布局原样恢复。**不破坏当前的空间布局**，就不需要重建它——这是"降低错误恢复成本"
在布局层面的体现。

### copy-mode

```tmux
setw -g mode-keys vi

bind-key -T copy-mode-vi 'v' send -X begin-selection
# "bind [" => vi mode, "space" => select, "y" => copy to system clipboard
bind -T copy-mode-vi y send -X copy-pipe-and-cancel "xclip -sel clip -i"
# 需要按回车后才退出选择模式
bind-key -T copy-mode-vi c send -X copy-pipe "xclip -sel clip -i"
```

`mode-keys vi` 让 copy-mode 用 Vim 键位（`hjkl`、`w`、`b`、`/` 搜索、`G`）——又一次
跨层一致。

`v` 开始选择、`y` 复制，跟 Vim 的 visual mode 完全对应。

注意 `y` 和 `c` 的区别：

- `y` = `copy-pipe-and-cancel` —— 复制并退出 copy-mode。
- `c` = `copy-pipe` —— 复制但**留在** copy-mode，可以继续选下一段。

**这是一对有意为之的双绑定**：单次复制用 `y`（一步到位），连续复制多段用 `c`
（不用反复进出）。给同一个操作提供"完成即退出"和"完成后继续"两个变体，是模态界面
里很有用的一个模式。

### 插件

```tmux
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'hmgle/tmux-recover'
set -g @tmux-recover-save-key 'C-s'
set -g @tmux-recover-restore-key 'C-r'
set -g @plugin 'Morantron/tmux-fingers'

set -g @plugin 'hmgle/tmux-nexus'
set -g @tmux_nexus_motion_key 's'
set -g @tmux_nexus_pane_key "/"
set -g @tmux_nexus_manager_key "F"
set -g @tmux_nexus_manager_fzf_options "-p -w 86% -h 58% -m"

bind-key "f" run-shell -b "~/.tmux/plugins/tmux-nexus/tnx manage pane switch"
bind-key "y" run-shell -b "~/.tmux/plugins/tmux-nexus/tnx manage clipboard"
```

四个插件，全部收在 prefix 之后：

| 键 | 插件 | 作用 |
| --- | --- | --- |
| `prefix C-s` / `prefix C-r` | tmux-recover | 保存/恢复会话布局 |
| `prefix s` | tmux-nexus | easymotion 式跳转 |
| `prefix /` | tmux-nexus | pane 内跳转 |
| `prefix F` / `prefix f` | tmux-nexus | fzf 选 pane |
| `prefix y` | tmux-nexus | 剪贴板管理 |

**"全部收在 prefix 之后"是关键。** 插件不占用任何 root key table，也就是说它们
一个键都没有从下游程序那里抢走。装 20 个插件也不会影响 Neovim 能绑什么键。

对比一下：如果这些插件用 `bind -n`（root 表，无需 prefix），每装一个就少一个可用
组合，而且冲突还很难发现。**前缀模式的最大价值就是命名空间隔离**——这是
[第六篇](06-who-gets-the-key.html) 原则二的实证。

## 状态可见性

这几行不是键位，但直接服务于"减少状态确认"：

```tmux
set -g window-style 'fg=#808080,bg=#101010'
set -g window-active-style 'fg=terminal,bg=terminal'
set -g pane-border-style 'fg=#808080,bg=#101010'
set -g pane-active-border-style 'fg=green,bg=terminal'
```

**非活动 pane 整体调暗，活动 pane 用终端原色，边框用绿色高亮。**

效果是：屏幕上分了四个 pane 时，你**不需要看光标在哪**——亮的那个就是活动 pane，
一眼可辨，不需要任何确认动作。

这跟 [第八篇](08-wezterm.html) 里关掉光标闪烁是同一套思路的两面：

- **重要状态要一眼可辨**（活动 pane 高亮）。
- **不重要的东西不要动**（光标不闪、动画降到 1fps）。

其余几条：

```tmux
setw -g monitor-activity on
set -g visual-activity on
set -g display-panes-time 8000
set-option -g history-limit 20000
```

`monitor-activity` 让后台 window 有输出时在状态栏标记——你不用挨个切过去看编译完了
没有。`display-panes-time 8000` 把 `prefix q`（显示 pane 编号）的停留时间从默认
1 秒延长到 8 秒，足够看清再决定。

## 环境变量

```tmux
# Refresh PATH from client environment when creating new sessions,
# so tools like nvm/pyenv won't get stale paths cached by tmux server.
set-option -ga update-environment " PATH"
```

[第六篇](06-who-gets-the-key.html) 详细讲过 tmux 的环境变量陷阱。这里只补一句：
`update-environment` 是把双刃剑——它能刷新变量，也能在 ssh attach 时**把变量标记为
移除**。加变量到这个列表之前，先想清楚"从 ssh attach 时这个变量缺失会怎样"。

## 小结

1. **prefix 选择要算代价**：`C-b` 抢走 Vim 翻页，`M-b` 抢走 readline 移词；无论选
   哪个，都配上 `bind <prefix> send-prefix` 让原功能可挽回。
2. **`escape-time 10` 是最重要的一行配置**，它消除的不是半秒延迟，是"我退出模式了
   吗"这个疑问。
3. **`extended-keys on` + `csi-u` 是 CSI u 链条的中间环节**，缺了它上下游都白配。
4. **`-c "#{pane_current_path}"` 让新 pane 继承目录**，消除的是问题本身而不是操作。
5. **插件全部收在 prefix 之后**，命名空间隔离让插件数量与下游可用键位解耦。
6. **非活动 pane 调暗**：重要状态一眼可辨，就不需要主动确认。

下一篇进入终端里的第一个程序：
[命令行层：zsh ZLE](10-zsh-zle.html)。
