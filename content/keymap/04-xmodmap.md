---
title: "Linux 按键体系 04：xmodmap 三层模型"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第四篇。
> [第二篇](02-key-encoding.html) 给出了四层编码，本篇聚焦其中的 X11 部分：
> `xmodmap` 到底在改什么，它的三层模型是什么，以及那个几乎所有人都踩过的求值时机
> 陷阱。

## 三层模型

`xmodmap` 看起来命令很多，实际上只操作三样东西：

```text
┌────────────────────────────────────────────────────┐
│ 1. keycode 表    keycode → keysym 列表             │
│                  keycode 66 = Escape               │
├────────────────────────────────────────────────────┤
│ 2. level 结构    keysym 列表内部按修饰键分档        │
│                  keycode 49 = grave asciitilde     │
│                              level1  level2        │
├────────────────────────────────────────────────────┤
│ 3. modifier 组   8 个组，每组挂若干 keysym          │
│                  shift lock control mod1 ... mod5  │
└────────────────────────────────────────────────────┘
```

先把当前状态导出来看：

```console
$ xmodmap -pke | head -5
keycode   8 =
keycode   9 = grave asciitilde grave asciitilde
keycode  10 = 1 exclam 1 exclam
keycode  11 = 2 at 2 at
keycode  12 = 3 numbersign 3 numbersign

$ xmodmap -pm
xmodmap:  up to 4 keys per modifier, (keycodes in parentheses):

shift       Shift_L (0x32),  Shift_R (0x3e)
lock        Caps_Lock (0x31)
control     Control_L (0x25),  Super_L (0x85),  Control_R (0x69)
mod1        Alt_L (0x40),  Alt_R (0x6c),  Meta_L (0xcd)
mod2        Num_Lock (0x4d)
mod3
mod4        Super_R (0x86),  Hyper_L (0xcf)
mod5        ISO_Level3_Shift (0x5c),  Mode_switch (0x8b)
```

`-pke` 是第 1、2 层，`-pm` 是第 3 层。

### 第 1 层：keycode → keysym

一行一个 keycode，等号右边是 keysym 列表。回忆
[第二篇](02-key-encoding.html) 的换算表：X keycode = 内核 keycode + 8，所以
`keycode 66` 是 Caps Lock 键，`keycode 9` 是 Esc 键。

### 第 2 层：level

keysym 列表里的位置有含义，前四个是：

```text
keycode 10 = 1 exclam 1 exclam
             │  │     │  └─ group 2, level 2
             │  │     └──── group 2, level 1
             │  └────────── group 1, level 2（按 Shift）
             └───────────── group 1, level 1（不按修饰键）
```

group 是键盘布局组（用 `setxkbmap` 切换多国布局时用），level 由修饰键选择。选 level 2
的是 Shift，选 level 3/4 的是 `ISO_Level3_Shift`（AltGr）。

只写一个 keysym 时，X 会按规则补全。比如 `keycode 66 = Escape` 意味着无论按不按
Shift 都是 `Escape`。

### 第 3 层：modifier 组

X11 有 8 个修饰键组，其中三个有固定语义（`shift`、`lock`、`control`），另外五个
（`mod1`–`mod5`）由约定分配：

| 组 | 通常是 |
| --- | --- |
| `shift` | Shift |
| `lock` | Caps Lock |
| `control` | Ctrl |
| `mod1` | Alt |
| `mod2` | Num Lock |
| `mod3` | （空） |
| `mod4` | Super / Windows 键 |
| `mod5` | AltGr / ISO_Level3_Shift |

**AwesomeWM 配置里的 `modkey = "Mod1"` 指的就是这个 `mod1` 组**，而不是某个物理键。
这个间接层是第 7 篇里理解"为什么我的 modkey 在物理 Windows 键上"的关键。

## 关键陷阱：add 和 remove 的求值时机不同

这是 `xmodmap` 最容易出错的地方，直接引用 man page：

```text
add MODIFIERNAME = KEYSYMNAME ...
    This adds all keys containing the given keysyms to the indicated
    modifier map.  The keysym names are evaluated after all input
    expressions are read to make it easy to write expressions to swap keys.

remove MODIFIERNAME = KEYSYMNAME ...
    This removes all keys containing the given keysyms from the indicated
    modifier map.  Unlike add, the keysym names are evaluated as the line
    is read in.
```

翻译成人话：

- **`remove` 立即求值**：读到这一行时，用**当时**的 keycode 表把 keysym 解析成
  keycode。
- **`add` 延迟到文件全部读完才求值**：用**最终**的 keycode 表解析。

这个不对称是故意设计的，为了让"交换两个键"能写得自然。但它意味着：

> **`add control = Super_L` 加进去的是哪个物理键，取决于文件末尾谁产生 `Super_L`，
> 而不是写这一行时谁产生 `Super_L`。**

而且要记住：**modifier 组绑定的是 keysym，不是 keycode**。你改了 keycode 表，
modifier 组的实际成员也会跟着变。

## 实战一：三键轮换

最简单的例子，`~/.local/bin/keymap_ng.sh`：

```sh
#!/bin/sh

# swap Esc CPAS_LOCK grave key
xmodmap -e "clear Lock"
xmodmap -e "keycode 66 = Escape"
xmodmap -e "keycode 49 = Caps_Lock"
xmodmap -e "keycode 9 = grave asciitilde"
xmodmap -e "add Lock = Caps_Lock"
```

三个键循环换位：

```text
物理 Caps Lock (66) ──▶ Escape
物理 `        (49) ──▶ Caps_Lock
物理 Esc       (9) ──▶ grave / asciitilde
```

为什么要 `clear Lock` 开头、`add Lock = Caps_Lock` 结尾？因为 Lock 组原本挂着
keycode 66（那时它产生 `Caps_Lock`）。执行 `keycode 66 = Escape` 之后，Lock 组会
指向一个不再产生 `Caps_Lock` 的键——状态不一致。所以先清空，最后按新表重新绑定。

注意这里每条 `xmodmap -e` 是独立进程，求值时机的问题被规避了。写成文件时就不一样了。

**这个改动只动了第 1 层。** `showkey -k` 按 Caps 还是 58，`xev` 里还是
`keycode 66`，变的只有 keysym。

## 实战二：把修饰键搬到拇指底下

真正体现设计意图的是 `~/.Xmodmap`：

```
remove mod1 = Super_L
add control = Super_L
! swap Esc CPAS_LOCK grave key
clear Lock
keycode 66 = Escape
keycode 49 = Caps_Lock
keycode 9 = grave asciitilde
add Lock = Caps_Lock
! swap Alt_L Super_L
keycode 64 = Super_L
keycode 133 = Alt_L
remove control = Alt_L
add mod1 = Alt_L
```

用求值规则展开一遍。文件读完后，keycode 表是：

```text
keycode   9 = grave asciitilde     ← 物理 Esc
keycode  49 = Caps_Lock            ← 物理 `
keycode  64 = Super_L              ← 物理左 Alt
keycode  66 = Escape               ← 物理 Caps Lock
keycode 133 = Alt_L                ← 物理左 Super（Windows 键）
```

然后三条 `add` 用这张最终表求值：

| 语句 | 最终产生该 keysym 的 keycode | 效果 |
| --- | --- | --- |
| `add control = Super_L` | 64 | **物理左 Alt 键 → Ctrl 修饰键** |
| `add Lock = Caps_Lock` | 49 | 物理 `` ` `` 键 → Caps Lock |
| `add mod1 = Alt_L` | 133 | **物理 Windows 键 → mod1（Alt）** |

于是键盘变成这样：

```text
        实际键位（ThinkPad 底排）
┌──────┬──────┬───────┬───────┬─────────────┐
│ Ctrl │  Fn  │ Super │  Alt  │    Space    │  ← 键帽印的
├──────┼──────┼───────┼───────┼─────────────┤
│ Ctrl │  Fn  │  Alt  │ Ctrl  │    Space    │  ← 实际行为
└──────┴──────┴───────┴───────┴─────────────┘
                  ▲       ▲
                  │       └─ mod1 的位置搬到了 Windows 键
                  └───────── 拇指最容易够到的位置给了 Ctrl
```

**设计意图很清楚**：ThinkPad 上紧挨空格键左边的那个键（键帽印 Alt）是拇指自然落点，
而 Ctrl 是终端环境里使用频率最高的修饰键——`Ctrl+a/e/w/u/r`（readline）、
`Ctrl+c/d/z`（信号）、`Ctrl+w/hjkl`（Neovim 窗口）、`Ctrl+Shift+Space`（WezTerm
leader）。把 Ctrl 放到拇指底下，右手小指就不用再去够左下角那个远端的 Ctrl 键。

而 Windows 键平时几乎没用，正好拿来当窗口管理器的 modkey。这样窗口管理层和终端层
在物理上就分开了——**这是第 6 篇"分层不冲突"原则的物理体现**。

### 这个文件的脆弱之处

`remove mod1 = Super_L`（第一行）在标准布局下是空操作：那时 `Super_L` 在 keycode
133 上，而 133 属于 `mod4` 不是 `mod1`。同样 `remove control = Alt_L` 也是空操作。

也就是说，这个文件**依赖初始状态恰好是标准布局**。如果它被执行两次，或者初始状态
不标准，结果就不可预期——第二次执行时 keycode 64 已经产生 `Super_L` 了，第一行
`remove mod1 = Super_L` 就不再是空操作。

`xmodmap` 不是幂等的。这一点值得记住。

## 实战三：清空重建的稳健写法

`dotconfig/bin/keymap.sh` 是同一件事的稳健版本：

```sh
#!/bin/sh

# swap Esc CPAS_LOCK grave key
xmodmap -e "clear Lock"
xmodmap -e "keycode 66 = Escape"
xmodmap -e "keycode 49 = Caps_Lock"
xmodmap -e "keycode 9 = grave asciitilde"
xmodmap -e "add Lock = Caps_Lock"
xmodmap -e "keycode 135 ="          # 禁用 Menu 键

xmodmap -e "keycode 64 = Super_L"   # 物理 Alt  → Super_L
xmodmap -e "keycode 133 = Alt_L"    # 物理 Win  → Alt_L

# 全部清空，不依赖初始状态
xmodmap -e "clear control"
xmodmap -e "clear mod1"
xmodmap -e "clear mod2"
xmodmap -e "clear mod3"
xmodmap -e "clear mod4"
xmodmap -e "clear mod5"

# 显式重建
xmodmap -e "add control = Control_L"
xmodmap -e "add control = Super_L"    # ← 物理 Alt 键
xmodmap -e "add control = Control_R"

xmodmap -e "add mod1 = Alt_L"         # ← 物理 Win 键
xmodmap -e "add mod1 = Alt_R"
xmodmap -e "add mod1 = Meta_L"

xmodmap -e "add mod2 = Num_Lock"
xmodmap -e "add mod4 = Super_R"
xmodmap -e "add mod4 = Hyper_L"
xmodmap -e "add mod5 = ISO_Level3_Shift"
xmodmap -e "add mod5 = Mode_switch"
```

差别在于它**先 `clear` 掉全部五个 mod 组再显式重建**。这样无论初始状态如何、执行
几次，结果都一样。

如果你要写自己的 `xmodmap` 配置，采用这个模式：**先改 keycode 表，再清空所有
modifier 组，最后显式重建。** 别依赖初始状态，别依赖 `remove`。

## 实战四：多把键盘

我有两把键盘：ThinkPad 内置的和外接的 Poker II。它们物理布局不同，需要不同的映射。

三份配置：

```
~/.Xmodmap        ← 通用/内置
~/.pokerXmodmap   ← Poker II
~/.tpXmodmap      ← ThinkPad（Poker 拔掉后恢复用）
```

`~/.tpXmodmap` 只做两键交换（Poker II 上 Esc 和 `` ` `` 的相对位置跟 ThinkPad 不同）：

```
! swap Esc CPAS_LOCK key
clear Lock
keycode 66 = Escape
keycode 9 = Caps_Lock
keycode 49 = grave asciitilde
add Lock = Caps_Lock
```

登录时按当前接的键盘选一份，`~/.xinitrc`：

```sh
#!/bin/sh

if [ -e /dev/input/by-id/usb-Heng_Yu_Technology_Poker_II-event-kbd ]; then
	/usr/bin/xmodmap $HOME/.pokerXmodmap
	exit 0
fi
if [ -s $HOME/.Xmodmap ]; then
	/usr/bin/xmodmap $HOME/.Xmodmap
fi
```

热插拔用 udev 规则触发：

```
# /etc/udev/rules.d/80-keyboard.rules
ATTR{idVendor}=="0f39", ATTR{idProduct}=="0611", ACTION=="add", RUN+="/home/gle/bin/pokerkeyboard_in.sh"
ACTION=="remove", SUBSYSTEM=="usb", RUN+="/home/gle/bin/pokerkeyboard_out.sh"
```

对应的脚本：

```sh
# pokerkeyboard_in.sh
/bin/sleep 0.2 && sudo -u gle /usr/bin/xmodmap -display :0 /home/gle/.pokerXmodmap >/dev/null 2>&1 &

# pokerkeyboard_out.sh
/bin/sleep 0.2
if [ ! -e /dev/input/by-id/usb-Heng_Yu_Technology_Poker_II-event-kbd ]; then
    sudo -u gle /usr/bin/xmodmap -display :0 /home/gle/.tpXmodmap >/dev/null 2>&1 &
fi
```

**这套东西能跑，但它是一堆补丁。** 注意里面的味道：

- `sleep 0.2` —— 等 X server 处理完设备添加。这是猜的，不是同步的。
- `-display :0` —— 硬编码 display。
- `sudo -u gle` —— udev 以 root 运行，要切回用户。
- remove 规则匹配所有 USB 设备移除事件，再靠脚本里判断设备节点存不存在。
- 拔掉键盘要恢复成 `.tpXmodmap`，等于维护两份逆操作。

**根本问题是 `xmodmap` 工作在错误的层次。** 它改的是 X server 的全局键盘映射，而
X server 在设备热插拔时会重建这张映射表，把你的设置冲掉。于是你只能在事后打补丁。

## xmodmap 的边界

总结它做不到的事：

| 限制 | 说明 |
| --- | --- |
| **热插拔失效** | X 重建键盘映射时冲掉设置，只能靠 udev 补救 |
| **Wayland 完全无效** | Wayland 下 keycode→keysym 翻译在客户端做 |
| **tty 无效** | 控制台走内核 keymap，跟 X 无关 |
| **不是幂等的** | `add`/`remove` 求值时机不同，重复执行结果不同 |
| **无法区分键盘** | 改的是全局映射，两把键盘只能用同一套 |
| **没有 tap/hold** | 无法做"轻点是 Esc，按住是 Ctrl" |
| **没有分层** | 无法做"按住空格时 hjkl 变方向键" |
| **是 XKB 的兼容层** | 现代 X 底层是 XKB，`xmodmap` 的改动被翻译成 XKB 操作，某些组合会出现意外行为 |

最后一条特别值得注意：X.Org 早就转向 XKB 了，`xmodmap` 只是保留的兼容接口。用
`setxkbmap -option ctrl:nocaps` 这类 XKB 选项通常比 `xmodmap` 更稳，但 XKB 的自定义
规则文件写起来门槛高得多。

而 tap/hold 和分层这两项，是最近十年键位设计里收益最大的两个特性——它们能让你**在不
增加组合键复杂度的前提下**扩展可用键位。这正是下一篇要下沉到内核层的原因。

## 调试速查

```console
# 当前 keycode → keysym 表
$ xmodmap -pke

# 当前 modifier 组
$ xmodmap -pm

# 实时看按键（keycode + keysym + 修饰键状态）
$ xev -event keyboard

# 查某个 keysym 在哪个 keycode 上
$ xmodmap -pke | rg 'Super_L'

# XKB 侧的完整状态（比 xmodmap 更权威）
$ setxkbmap -print -verbose 10
$ xkbcomp :0 - | less

# 恢复默认（重新加载 XKB）
$ setxkbmap -layout us
```

一个实用技巧：改键改乱了、又不想重登录，`setxkbmap -layout us` 会重新加载 XKB 规则，
把 `xmodmap` 的改动全部冲掉，等于软复位。

## 小结

1. **三层模型**：keycode→keysym 表、keysym 列表的 level 结构、8 个 modifier 组。
2. **`add` 延迟求值、`remove` 立即求值**，且 modifier 组绑的是 keysym 不是 keycode。
   这两条解释了几乎所有"xmodmap 结果和预期不一样"。
3. **稳健写法**：先改 keycode 表 → `clear` 所有 mod 组 → 显式重建。别依赖初始状态。
4. **把 Ctrl 搬到拇指位置**是这套配置的核心设计，它让终端层最高频的修饰键变得几乎
   零成本；把 modkey 搬到 Windows 键则把窗口层和终端层在物理上分开。
5. **xmodmap 的天花板**是热插拔、Wayland、tty，以及没有 tap/hold 和分层。

下一篇把映射下沉到内核层：
[到内核层重塑键盘：keyd、udev 与热插拔](05-keyd.html)。
