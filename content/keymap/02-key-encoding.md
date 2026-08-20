---
title: "Linux 按键体系（二）：四层编码"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第二篇。
> [上一篇](01-key-journey.html) 走完了一次按键的完整链路，本篇聚焦其中最容易搞混的
> 部分：**同一个物理键，在每一层叫什么名字，谁负责翻译，改哪一层影响到哪里。**

## 一个键的五个身份

我的键盘上 `Caps Lock`、`Esc`、`` ` `` 这三个键被轮换过：Caps 变 Esc，Esc 变反引号，
反引号变 Caps。用这个配置做例子最清楚——因为它让"名字"和"位置"彻底脱钩了。

按下**物理上的 Caps Lock 键**，它在各层的身份是：

| 层 | 名字 | 值 |
| --- | --- | --- |
| PS/2 协议 | scancode (set 1) | `0x3a` |
| 内核 input | keycode | `KEY_CAPSLOCK` = `58` |
| X11 | X keycode | `66` |
| XKB（默认） | keysym | `Caps_Lock` (`0xffe5`) |
| XKB（我的配置） | keysym | `Escape` (`0xff1b`) |
| 终端 | 字节 | `0x1b` |

从上往下，每一层做一次翻译，每一层都可以被单独改写。理解这四层编码，等于拿到了改键
的全部自由度。

## 第一层：scancode —— 硬件协议的方言

scancode 是键盘控制器发出的原始编号，**它是协议相关的**，不同接口完全不是一套。

### PS/2 / 内置键盘

AT 键盘定义了三套 scancode set，实际上常用的是 set 1 和 set 2。协议上区分：

- **make code**：按下时发送。
- **break code**：抬起时发送。set 1 里是 make code 加 `0x80`；set 2 里是前缀 `0xF0`
  再跟 make code。
- **`0xE0` 前缀**：扩展键。原始 IBM PC 键盘没有右 Ctrl、右 Alt、方向键这些，后来
  加的键用 `0xE0` 前缀跟一个已有的 make code 表示。这就是为什么左右 Ctrl 在硬件层
  能区分。

笔记本内置键盘默认工作在 set 2，但 i8042 控制器通常开着 translation，把 set 2 转成
set 1 再交给内核——所以设备名叫 "AT Translated Set 2 keyboard"：

```console
$ rg -A3 'AT Translated' /proc/bus/input/devices
N: Name="AT Translated Set 2 keyboard"
P: Phys=isa0060/serio0/input0
S: Sysfs=/devices/platform/i8042/serio0/input/input0
```

### USB 键盘

USB 键盘不用 scancode，用 **HID Usage**。键盘的 Usage Page 是 `0x07`，每个键一个
Usage ID：`a` = `0x04`，`Esc` = `0x29`，`Caps Lock` = `0x39`。

内核在 `EV_MSC/MSC_SCAN` 事件里报告的"scancode"，对 USB 设备来说是
`(usage_page << 16) | usage_id`，也就是 Escape 会显示成 `0x70029`。看到这个数字不要
慌，它就是 HID usage 拼出来的。

### 观测

```console
$ sudo showkey -s
按 Caps Lock：
0x3a 0xba          ← make / break（set 1）
```

`showkey` 要在真实 tty（`Ctrl+Alt+F2`）里跑。

**这一层几乎不需要你去改。** 唯一的例外是某些笔记本的特殊功能键内核不认识，需要用
`setkeycodes` 或 udev hwdb 手动补一条 scancode → keycode 映射。

## 第二层：keycode —— 第一个统一命名空间

内核驱动把 scancode 翻译成 **keycode**，这是全链路第一个跟硬件无关的编号，定义在
`include/uapi/linux/input-event-codes.h`：

```c
#define KEY_ESC          1
#define KEY_GRAVE       41
#define KEY_LEFTCTRL    29
#define KEY_LEFTALT     56
#define KEY_CAPSLOCK    58
#define KEY_LEFTMETA   125
#define KEY_COMPOSE    127
```

不管你插的是 ThinkPad 内置键盘还是 Poker II，`Caps Lock` 到这里都是 `58`。

### X11 的 +8 偏移

X server 读 evdev 之后要再加 8：

```text
X keycode = 内核 keycode + 8
```

原因是历史遗留：X11 协议规定 keycode 范围是 8–255，`0`–`7` 保留。于是有了这张
**读配置文件必备对照表**：

| 键 | 内核 keycode | X keycode |
| --- | --- | --- |
| `Esc` | 1 | **9** |
| `` ` `` | 41 | **49** |
| 左 `Ctrl` | 29 | 37 |
| 左 `Alt` | 56 | **64** |
| `Caps Lock` | 58 | **66** |
| 左 `Super` | 125 | **133** |
| `Menu` | 127 | **135** |

加粗的几个正是我 `.Xmodmap` 里出现的数字。下一次在别人的配置里看到 `keycode 133`，
你应该立刻反应过来：那是左边的 Windows 键。

### 观测

```console
$ sudo showkey -k
keycode  58 press
keycode  58 release
```

X11 下用 `xev`：

```console
$ xev -event keyboard
KeyPress event, ..., keycode 66 (keysym 0xff1b, Escape), ...
```

### 这一层能改什么

改 keycode 映射的工具有 `setkeycodes`、udev hwdb、以及 **keyd**。它们的共同特点是
**改动发生在内核层，对上面所有层透明**——X11、Wayland、tty 全都生效，而且不受热插拔
影响。这是第 5 篇的主题。

## 第三层：keysym —— 从"哪个键"到"什么含义"

keycode 只说"66 号键"，不说这个键**是什么意思**。赋予含义的是 **keysym**，定义在
`/usr/include/X11/keysymdef.h`：

```c
#define XK_Escape             0xff1b  /* U+001B ESCAPE */
#define XK_grave              0x0060  /* U+0060 GRAVE ACCENT */
#define XK_asciitilde         0x007e  /* U+007E TILDE */
#define XK_Caps_Lock          0xffe5  /* Caps lock */
#define XK_Control_L          0xffe3  /* Left control */
#define XK_ISO_Level3_Shift   0xfe03
```

值得注意的规律：

- **可打印字符的 keysym 直接等于它的 ASCII / Unicode 码点**（`grave` = `0x60`）。
- **功能键和修饰键在 `0xff00`–`0xffff` 区间**，它们没有对应字符。
- **`XF86*` 系列**（`XF86AudioRaiseVolume` 等）是厂商扩展，多媒体键用的就是它们。

**keysym 不等于字符。** `Control_L` 是一个 keysym，但它不产生任何字符；
`ISO_Level3_Shift`（就是 AltGr）也是。这个区别在下一层会变得很重要。

### 一个 keycode 对应多个 keysym

XKB 把每个 keycode 映射到一个 **keysym 列表**，按 group（键盘布局组）和 level
（修饰键档位）组织。最常见的是前两个 level：

```text
keycode 49 = grave asciitilde
             │      └─ level 2：按住 Shift
             └─ level 1：不按修饰键
```

选哪个 level 由修饰键状态决定。这也是为什么 `Shift` 不是"另一个按键"，而是**一个
改变翻译规则的状态位**。

`ISO_Level3_Shift`（AltGr）就是用来选 level 3 和 4 的，欧洲键盘靠它输入 `€`、`ä`
之类。我的配置里把它挂在 `mod5` 上，虽然平时用不到。

### 观测

```console
# 完整的 keycode → keysym 表
$ xmodmap -pke | head
keycode   9 = grave asciitilde grave asciitilde
keycode  49 = Caps_Lock
keycode  66 = Escape

# 修饰键组
$ xmodmap -pm
shift       Shift_L (0x32),  Shift_R (0x3e)
lock        Caps_Lock (0x31)
control     Control_L (0x25),  Super_L (0x85),  Control_R (0x69)
mod1        Alt_L (0x40),  Alt_R (0x6c)
```

### 这一层能改什么

`xmodmap` 和 XKB 改的就是这一层：keycode 不变，改它翻译成什么 keysym，以及哪些
keysym 算作修饰键。**这是第 4 篇的主题。**

关键限制：这一层是 **X server 的全局状态**。Wayland 下翻译发生在客户端，
`xmodmap` 完全无效；tty 里也无效。

## 第四层：字符与字节

keysym 有了，应用程序最终要拿到的是**字符**（GUI 程序）或**字节序列**（终端程序）。

GUI 程序比较直接：GTK/Qt 通过输入法框架把 keysym + 修饰键状态转成 Unicode 字符串。

终端程序要复杂得多。终端模拟器必须把 keysym 编码成字节写进 PTY：

| keysym | 修饰键 | 字节 |
| --- | --- | --- |
| `a` | — | `0x61` |
| `a` | Shift | `0x41`（`A`） |
| `a` | Ctrl | `0x01` |
| `b` | Alt | `0x1b 0x62`（`ESC b`） |
| `Escape` | — | `0x1b` |
| `Up` | — | `ESC [ A` 或 `ESC O A` |
| `Control_L` | — | **什么也不发** |

最后一行是重点：**修饰键本身不产生任何字节**。它只是改变别的键的编码方式。这是
终端程序无法绑定"单独按 Ctrl"的根本原因，也是第 12 篇里"rime 收不到单独的 `Alt_L`"
那个坑的来源。

这一层的编码规则本身就是一个大话题，[第 3 篇](03-terminal-keys.html) 专门讲。

## 把四层串起来：我的三键轮换

现在回头看 `~/.local/bin/keymap_ng.sh`：

```sh
#!/bin/sh

# swap Esc CPAS_LOCK grave key
xmodmap -e "clear Lock"
xmodmap -e "keycode 66 = Escape"
xmodmap -e "keycode 49 = Caps_Lock"
xmodmap -e "keycode 9 = grave asciitilde"
xmodmap -e "add Lock = Caps_Lock"
```

用刚才那张对照表翻译一遍：

```text
keycode 66（物理 Caps Lock 键） = Escape
keycode 49（物理 ` 键）         = Caps_Lock
keycode  9（物理 Esc 键）       = grave asciitilde
```

三个键循环换位。注意它**只动了第三层**：

- scancode 没变——`showkey -s` 按 Caps 还是 `0x3a`。
- keycode 没变——`showkey -k` 按 Caps 还是 `58`，`xev` 里还是 `keycode 66`。
- 变的只有 keysym：`66` 现在翻译成 `Escape`。

而 `clear Lock` / `add Lock = Caps_Lock` 这两行动的是 **modifier 组**：先把 Lock 组
清空，最后重新把 `Caps_Lock` 这个 keysym 加回 Lock 组。为什么要清空再加？因为中间
`keycode 66 = Escape` 那一步，如果 Lock 组还绑着旧的 keycode 66，X server 会处于
一个尴尬状态：Lock 修饰键指向一个已经不产生 `Caps_Lock` keysym 的键。

**修饰键组绑定的是 keysym，不是 keycode。** 这个细节是 `xmodmap` 配置里最常见的
坑，第 4 篇会展开。

## 改哪一层：决策表

这是本篇最实用的产出。想改键时，先问自己"我要影响的范围有多大"：

| 需求 | 改哪一层 | 工具 | 生效范围 |
| --- | --- | --- | --- |
| 内核不认识某个特殊功能键 | scancode → keycode | `setkeycodes`、udev hwdb | 全系统 |
| Caps 换 Ctrl / Esc，希望到处都生效 | keycode | **keyd** | tty + X + Wayland，含热插拔 |
| 需要 tap/hold 双功能、分层 | keycode | **keyd** | 同上 |
| 只想在 X 下换个 keysym | keysym | `xmodmap`、`setxkbmap` | 仅当前 X server |
| 换整套键盘布局（如 Dvorak） | keysym | XKB / `setxkbmap` | 仅当前 X server |
| 全局快捷键（启动程序、切窗口） | 第 4 层 | 窗口管理器 | 所有应用 |
| 终端里的组合键 | 第 6 层 | 终端 / tmux / zsh / nvim | 该程序内 |

一条经验：**能在低层解决的，尽量在低层解决**。Caps Lock 换 Ctrl 这种事，放在 keyd
里一次配好，tty、X、Wayland、ssh 进来的会话全都一致；放在 `xmodmap` 里，你要处理
热插拔、要处理 Wayland、要处理"为什么 tty 里不生效"。

反过来，**能在高层解决的，不要往低层塞**。把"打开终端"绑到 keyd 的某个 layer 上是
可以做到的，但那意味着这个键在任何程序里都被吃掉了，包括你临时需要它的场合。

## 小结

1. **四层编码**：scancode（协议相关）→ keycode（内核统一）→ keysym（含义）→
   字符/字节（应用能用的形式）。
2. **X keycode = 内核 keycode + 8**，读任何 `xmodmap` 配置都要先做这个换算。
3. **一个 keycode 对应多个 keysym**，由修饰键选 level；修饰键本身不产生字节。
4. **修饰键组绑定的是 keysym 而非 keycode**，这是 `xmodmap` 最容易踩的坑。
5. 改键先决定层次：**低层影响广而统一，高层影响窄而灵活**。

下一篇进入终端：为什么 `Ctrl+i` 和 `Tab` 是同一个键，以及现代终端怎么解决它——
[终端里的按键：转义序列、terminfo 与 CSI u](03-terminal-keys.html)。
