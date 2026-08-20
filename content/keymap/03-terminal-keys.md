---
title: "Linux 按键体系（三）：终端里的按键"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第三篇。
> [上一篇](02-key-encoding.html) 讲到最后一层翻译是"keysym → 字节"。本篇把这层
> 讲透：终端到底怎么编码按键，为什么这套编码是有损的，以及现代终端怎么修补它。
>
> **这一篇是整个系列里约束性最强的一篇。** 后面第 9–11 篇里 tmux、zsh、Neovim
> 能绑什么键、不能绑什么键，全部由这里的规则决定。

## 终端只能收到字节

先明确前提：终端里运行的程序（zsh、tmux、Neovim）**看不到按键事件**。它们从 PTY
读到的只是一串字节。所有关于"用户按了什么"的信息，都必须编码进这串字节里。

而这套编码方案是 1970 年代为 VT100 之类的物理终端设计的，当时的设计目标是"用 7 位
ASCII 表达一切"。今天我们要用它表达 `Ctrl+Shift+F5`，捉襟见肘是必然的。

## 规则一：Ctrl 是一次位运算

ASCII 表里 `0x00`–`0x1f` 是 33 个控制字符。终端把 `Ctrl+字母` 编码成：

```text
byte = ASCII(letter) & 0x1f
```

也就是"把第 6、7 位清零"。于是：

| 组合 | 计算 | 字节 | 撞上了谁 |
| --- | --- | --- | --- |
| `Ctrl+@` | `0x40 & 0x1f` | `0x00` | NUL |
| `Ctrl+c` | `0x63 & 0x1f` | `0x03` | ETX（行规程转 SIGINT） |
| `Ctrl+h` | `0x68 & 0x1f` | `0x08` | **Backspace** |
| `Ctrl+i` | `0x69 & 0x1f` | `0x09` | **Tab** |
| `Ctrl+j` | `0x6a & 0x1f` | `0x0a` | **LF（换行）** |
| `Ctrl+m` | `0x6d & 0x1f` | `0x0d` | **CR（回车）** |
| `Ctrl+[` | `0x5b & 0x1f` | `0x1b` | **Esc** |

**这就是所有"这两个键绑不开"问题的根源。** 不是终端偷懒，是这两个组合在字节层面
本来就是同一个东西。传统终端下：

- 在 Neovim 里 `<C-i>` 和 `<Tab>` 无法分别映射（而 `<C-i>` 默认是 jumplist 前进，
  所以给 `<Tab>` 绑东西会连带影响它）。
- `<C-m>` 和 `<CR>` 是同一个键。
- `<C-[>` 和 `<Esc>` 是同一个键——这一点在第 12 篇里其实是**好事**：为 Esc 写的
  输入法切换逻辑天然覆盖了 `Ctrl+[`。

顺带一提，大小写在这里被抹掉了：`Ctrl+a` 和 `Ctrl+Shift+a` 都是 `0x01`。所以
**传统终端下 `Ctrl+Shift+字母` 这一整类组合根本不存在**。

## 规则二：特殊键用转义序列

没有对应控制字符的键（方向键、功能键、Home/End）用**转义序列**表示，格式是
`ESC` + 一串字符。最常见的是 CSI（Control Sequence Introducer）系列，即 `ESC [`：

| 按键 | 字节 | 可读形式 |
| --- | --- | --- |
| `↑` | `1b 5b 41` | `ESC [ A` |
| `↓` | `1b 5b 42` | `ESC [ B` |
| `Home` | `1b 5b 31 7e` | `ESC [ 1 ~` |
| `F1` | `1b 4f 50` | `ESC O P` |

自己看一眼的办法：

```console
$ cat -v
^[[A       ← 上箭头
^[[1~      ← Home
```

或者在终端里按 `Ctrl+v` 再按目标键（`Ctrl+v` 是"下一个键按原样插入"）。

### 同一个键有两种编码：application mode

麻烦的是，方向键有**两套**编码，取决于终端处于哪种模式：

| 模式 | `↑` 发送 |
| --- | --- |
| Normal cursor keys | `ESC [ A` |
| Application cursor keys（DECCKM） | `ESC O A` |

全屏程序启动时通常会发 `ESC [ ? 1 h` 切到 application 模式，退出时发
`ESC [ ? 1 l` 切回来。terminfo 里对应 `smkx` / `rmkx`：

```console
$ infocmp -1 tmux-256color | rg 'kcuu1|smkx|rmkx'
	kcuu1=\EOA,
	rmkx=\E[?1l\E>,
	smkx=\E[?1h\E=,
```

`kcuu1=\EOA` —— 注意这里记的是 **application 模式**的序列。

**这直接影响 zsh 配置的正确写法。** 如果你在 `.zshrc` 里硬编码：

```zsh
bindkey '^[[A' up-line-or-search      # 只在 normal 模式下对
```

那么当 zsh 处于 application 模式时这条绑定就不生效。正确做法是从 terminfo 里取：

```zsh
[[ -n "${terminfo[kcuu1]}" ]] && bindkey "${terminfo[kcuu1]}" up-line-or-search
```

这样换终端、换 `TERM` 都不会坏。第 10 篇会再回到这个话题。

## 规则三：Alt 是 ESC 前缀，而且有歧义

`Alt+b` 的传统编码是 `ESC` + `b`，也就是 `1b 62`。

问题来了：**这跟"先按 Esc，再按 b"的字节序列完全一样**。程序无法从字节上区分
"Alt+b" 和 "Esc 然后 b"。

唯一的线索是**时间**：`Alt+b` 的两个字节几乎同时到达，手动按则中间有间隔。所以每个
支持 Alt 的终端程序都有一个超时参数：

```tmux
# .tmux.conf
# Set the time in milliseconds for which tmux waits after an escape is input
# to determine if it is part of a function or meta key sequences. The default
# is 500 milliseconds.
set-option -s escape-time 10
```

tmux 默认 500ms 意味着：**在 tmux 里按 Esc 之后，要等半秒它才确定你不是在按
Alt 组合**。对 Vim 用户来说这是灾难——每次从插入模式退出都卡半秒。把它调到 10ms
是几乎所有 tmux + Vim 用户的第一条配置。

Neovim 那边对应的是 `ttimeoutlen`。

调到 10ms 的代价是：如果你通过高延迟的 ssh 连接操作，`ESC` 和 `b` 两个字节可能被
拆进不同的 TCP 包，间隔超过 10ms，于是 `Alt+b` 被识别成 `Esc` 然后 `b`。这是一个
真实的取舍，不是"调到最小就对了"。

## 规则四：带修饰键的特殊键

`Shift+↑` 怎么表示？xterm 定义了一个扩展格式：

```text
ESC [ 1 ; <modifier> A
```

`<modifier>` 是一个位图加一：

```text
modifier = 1 + (Shift ? 1 : 0) + (Alt ? 2 : 0) + (Ctrl ? 4 : 0) + (Super ? 8 : 0)
```

于是：

| 组合 | modifier | 序列 |
| --- | --- | --- |
| `Shift+↑` | 1+1 = 2 | `ESC [ 1 ; 2 A` |
| `Alt+↑` | 1+2 = 3 | `ESC [ 1 ; 3 A` |
| `Ctrl+↑` | 1+4 = 5 | `ESC [ 1 ; 5 A` |
| `Ctrl+Shift+↑` | 1+4+1 = 6 | `ESC [ 1 ; 6 A` |

**注意这个机制只适用于本来就用转义序列的键**（方向键、功能键、Home/End）。普通字母
键仍然走规则一那套位运算，`Ctrl+Shift+h` 依然无法表达。

## 结论：传统终端下绑不了的键

把上面的规则汇总，传统终端下**不可能**区分或表达的组合：

- `Ctrl+i` / `Tab`
- `Ctrl+m` / `Enter`
- `Ctrl+[` / `Esc`
- `Ctrl+h` / `Backspace`
- `Ctrl+Shift+<字母>`（等同于 `Ctrl+<字母>`）
- `Shift+Enter`、`Ctrl+Enter`
- `Ctrl+,`、`Ctrl+.`、`Ctrl+;`、`Ctrl+/` 等（大部分标点没有控制字符对应）
- 任何"单独按下修饰键"的事件

最后一条值得单独强调：**修饰键本身不产生任何字节**。终端程序永远看不到"用户单独按了
一下 Alt"。第 12 篇里我尝试用 rime 的 `ascii_composer/switch_key/Alt_L` 来解决 tmux
prefix 问题，失败的根本原因就在这里。

## 现代解法：CSI u 与 kitty keyboard protocol

问题的本质是"信息在编码时丢了"。解法也就只有一个方向：**换一套不丢信息的编码**。

### CSI u（fixterms）

Paul LeoNerd Evans 提出的 [fixterms](http://www.leonerd.org.uk/hacks/fixterms/)
方案，格式极简：

```text
ESC [ <unicode-码点> ; <modifier> u
```

于是原本无法表达的组合有了唯一编码：

| 组合 | CSI u 序列 |
| --- | --- |
| `Ctrl+i` | `ESC [ 105 ; 5 u` |
| `Tab` | `0x09`（保持不变） |
| `Ctrl+Shift+h` | `ESC [ 104 ; 6 u` |
| `Ctrl+;` | `ESC [ 59 ; 5 u` |

`105` 是 `i` 的码点，`5` 是 `1 + Ctrl(4)`。因为码点和修饰键分开编码，**任意组合都能
无歧义表达**。

### kitty keyboard protocol

kitty 的方案是 CSI u 的超集，用渐进增强的方式协商能力：程序发
`ESC [ > <flags> u` 声明自己想要哪些特性（区分按下/抬起、上报修饰键事件、
上报所有键等），终端按能力应答。

它能做到 CSI u 做不到的事，比如**上报单独按下修饰键**——这对某些编辑器插件很有用。

### 我的配置

WezTerm 侧：

```lua
-- config/wezterm/wezterm.lua
disable_default_key_bindings = true,
enable_csi_u_key_encoding = true,
leader = { key = "Space", mods = "CTRL|SHIFT" },
```

tmux 侧：

```tmux
# .tmux.conf
set -g default-terminal "tmux-256color"
set -s terminal-features[100] "xterm*:RGB"
set -g extended-keys on
set -g extended-keys-format csi-u
```

注意 `leader = { key = "Space", mods = "CTRL|SHIFT" }` 这一行——`Ctrl+Shift+Space`
在传统编码下是不存在的组合，正因为开了 CSI u 才能用作 leader。这是个很划算的选择：
它不可能跟任何终端程序的既有键位冲突。

### 关键：整条链路都要支持

CSI u 有一个容易踩的坑：**它必须逐层贯通**。

```text
WezTerm 编码 CSI u ──▶ tmux 解码 ──▶ tmux 再编码 ──▶ Neovim 解码
   ✓                     ✓ 需要        ✓ 需要        ✓ 需要
                    extended-keys on   csi-u 格式    支持 CSI u
```

任何一环不支持，信息就在那里降级丢失。常见的失败场景：

- tmux 版本太老（`extended-keys` 是 3.2+ 加的）。
- ssh 到远端，远端的 tmux 没开 `extended-keys`。
- `TERM` 设置不对，导致中间某层用了保守的编码。

所以在实践中，我的 Neovim 键位设计仍然**假设 CSI u 可能不可用**：核心操作只用传统
编码能表达的组合，CSI u 只用来做锦上添花的绑定。跨机器、跨 ssh 的稳定性比多几个
可绑的键更重要。

## 验证方法

从下往上逐层确认，这是排查此类问题的标准流程：

```console
# 1. 终端到底发了什么字节
$ cat -v
# 或者在任何终端程序里按 Ctrl+v 再按目标键

# 2. WezTerm 自己怎么理解这个键
$ wezterm show-keys --lua | rg "key = 'b'"

# 3. tmux 收到了什么
$ tmux list-keys -T root | rg 'C-i'
# tmux 的 copy-mode 里按 Ctrl+v 也能看到原始输入

# 4. Neovim 收到了什么
:map <C-i>
# 或者在插入模式按 Ctrl+v 再按目标键

# 5. 当前 TERM 支持什么
$ infocmp -1 "$TERM" | rg 'kcuu1|smkx|rmkx'
```

## 对键位设计的影响

回到系列主线——**保护注意力、减少状态确认、降低错误恢复成本**。终端编码的限制在这
三条上都有直接后果：

**一、别跟历史包袱较劲。** `Ctrl+i` / `Tab` 这类冲突，硬要通过 CSI u 分开绑，代价是
配置在某台没配好的机器上会静默降级——你按 `Ctrl+i` 得到的是 `Tab` 的行为。**静默
降级比功能缺失更伤注意力**，因为你要花时间才能意识到"哦，这台机器不一样"。

**二、优先选那些"不可能被占用"的组合。** `Ctrl+Shift+Space` 作为 leader 就是这个
思路：传统编码下它不存在，所以没有任何历史约定占用它。

**三、`escape-time` 这类"隐形"设置的价值远大于多绑几个键。** 500ms 改成 10ms，
每次从插入模式退出省半秒——但真正的收益不是那半秒，是消除了"我到底退出没有"这个
状态疑问。

## 小结

1. **`Ctrl+字母` 是位运算 `& 0x1f`**，所以 `Ctrl+i`=`Tab`、`Ctrl+m`=`Enter`、
   `Ctrl+[`=`Esc`，且 `Ctrl+Shift+字母` 不存在。
2. **特殊键用转义序列**，且有 normal / application 两套；zsh 里要用
   `$terminfo[kcuu1]` 而不是硬编码。
3. **`Alt+x` 就是 `ESC x`**，只能靠超时区分；`escape-time 10` 是必备配置。
4. **修饰键本身不产生字节**，终端程序看不到"单独按了 Alt"。
5. **CSI u / kitty protocol 解决了信息丢失**，但必须终端、tmux、应用逐层贯通，
   否则会静默降级。

下一篇回到 X11，把 `xmodmap` 的三层模型和实战讲清楚：
[X11 三层模型：keycode / keysym / modifier 组](04-xmodmap.html)。
