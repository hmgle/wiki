---
title: "Linux 按键体系（一）：一次按键的旅程"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第一篇，回答系列的第一个核心
> 问题：**按下一个键，从物理层到应用层到底发生了什么？**
> 后面所有篇章——改键、冲突诊断、输入法联动——都建立在这条链路上。

## 从一个答不上来的问题说起

在我的机器上按一次 `Esc`，实际发生的事情是这样的：键帽下面的触点闭合，键盘控制器
扫描到了它，通过 USB 报上来一个数字 `0x29`；内核把它变成 `1`；X server 把它变成
`9`；XKB 把 `9` 解释成 keysym `grave`（是的，`grave`，不是 `Escape`，因为我把这三个
键轮换过）；终端把它编码成字节 `0x60`；tmux 转发；Neovim 收到一个反引号。

同一次物理动作，在不同的层有五个不同的名字。绝大多数"快捷键失灵"的问题，本质都是
你以为自己在跟某一层说话，实际上另一层先把这个键截走了。

所以先把整条链路拆开。

## 全景图

```text
┌─────────────────────────────────────────────────────────────┐
│ 0. 键盘内部     矩阵扫描 / 去抖 / MCU 编码                   │
│                 输出：USB HID Usage 或 PS/2 scancode         │
├─────────────────────────────────────────────────────────────┤
│ 1. 传输         USB HID Report  |  i8042 → PS/2 scancode     │
├─────────────────────────────────────────────────────────────┤
│ 2. 内核 input   scancode → keycode（KEY_ESC = 1）            │
│                 输出：/dev/input/eventN 上的 input_event     │
├──────────────┬──────────────────────────────────────────────┤
│ 3a. 控制台    │ 3b. X server / Wayland compositor            │
│  loadkeys     │  X keycode = kernel keycode + 8              │
│  内核 keymap  │  XKB: keycode → keysym（+ modifier 状态）    │
├──────────────┴──────────────────────────────────────────────┤
│ 4. 窗口管理器   AwesomeWM：XGrabKey 抢占的全局键在这里被吃掉 │
├─────────────────────────────────────────────────────────────┤
│ 5. 输入法       fcitx5：拦截 → 交给 rime → 吞掉 或 透传      │
├─────────────────────────────────────────────────────────────┤
│ 6. 终端模拟器   WezTerm：keysym → 字节序列（含转义序列）     │
│                 写入 PTY master                              │
├─────────────────────────────────────────────────────────────┤
│ 7. 行规程       termios：C-c → SIGINT、回显、行缓冲          │
├─────────────────────────────────────────────────────────────┤
│ 8. 终端内程序   tmux 解码 → 再编码 → 内层 PTY → zsh / Neovim │
└─────────────────────────────────────────────────────────────┘
```

下面逐层展开。

## 第 0 层：键盘内部

### 矩阵扫描

键盘不会给每个键单独拉一根线。104 个键如果一键一线，MCU 需要 104 个 IO 口，成本
和布线都不现实。实际做法是把键排成矩阵：比如 8 行 × 18 列，MCU 依次给每一行通电，
然后读所有列的电平，某一列有电就说明该行该列交叉点的键被按下了。这样 26 个 IO 口
就够了。

代价是**鬼键（ghosting）**。如果同时按下矩阵里构成一个矩形的三个角，电流会绕路，
MCU 会误判第四个角也被按下了。廉价键盘的解法是"封锁（blocking）"——检测到这种模式
就干脆全部丢弃；正经键盘的解法是给每个触点串一个二极管挡住反向电流，这就是
**NKRO（N-key rollover）** 的物理基础。

这解释了一个具体现象：有些薄膜键盘上 `Ctrl+Shift+某键` 的三键组合会静默失效，而且
只在特定几个键上失效。那不是软件问题，是矩阵布局决定的。

### 去抖

机械触点闭合时会在几毫秒内反复通断。固件必须做去抖（debounce），典型做法是检测到
状态变化后等 5ms 再确认。这是键盘固有延迟的主要来源之一，也是为什么"按一下出两个
字符"这种硬件故障通常意味着去抖时间不够或者触点老化。

### 编码

去抖之后，MCU 把"哪个键、按下还是抬起"编码成协议规定的格式：

- **USB 键盘**：按 HID（Human Interface Device）规范，键盘的 Usage Page 是 `0x07`，
  每个键有一个 Usage ID。`a` 是 `0x04`，`Esc` 是 `0x29`。注意这个编号跟 ASCII 毫无
  关系——HID 层根本不知道"字符"是什么，它只报告"7 号键被按下了"。
- **笔记本内置键盘 / PS/2**：走 i8042 控制器，用 scancode。AT 键盘默认使用
  scancode set 2，而 i8042 控制器通常开着 translation，把它转成 set 1 再交给内核。

后一句在我的机器上有直接证据：

```console
$ rg -A3 'AT Translated' /proc/bus/input/devices
N: Name="AT Translated Set 2 keyboard"
P: Phys=isa0060/serio0/input0
S: Sysfs=/devices/platform/i8042/serio0/input/input0
```

设备名里的 "Translated Set 2" 说的就是这件事。

## 第 1 层：传输

USB 键盘每次状态变化会上报一个 8 字节的 boot protocol report：第 1 字节是修饰键
位图（Ctrl/Shift/Alt/GUI 的左右共 8 位），第 2 字节保留，后 6 字节是当前按住的
普通键的 Usage ID。

这个格式有个有意思的推论：**boot protocol 最多同时报告 6 个普通键**。这就是所谓
6KRO 的来源。真正的 NKRO 键盘要用自定义 HID report descriptor（位图模式）才能突破，
而有些 BIOS 只认 boot protocol，所以 NKRO 键盘常常带一个切换开关。

另一个推论：**修饰键是单独一个字节的位图，不占那 6 个槽位**。所以
`Ctrl+Shift+Alt+某键` 这种组合在传输层从来不是问题，问题都出在更上层。

## 第 2 层：内核 input 子系统

内核驱动（USB 键盘走 `usbhid` + `hid-input`，内置键盘走 `atkbd`）拿到 scancode 或
HID usage 后，第一件事是查表转换成**内核 keycode**——一套跟硬件协议无关的统一编号，
定义在 `include/uapi/linux/input-event-codes.h`：

```c
#define KEY_ESC          1
#define KEY_1            2
#define KEY_GRAVE       41
#define KEY_LEFTCTRL    29
#define KEY_LEFTSHIFT   42
#define KEY_LEFTALT     56
#define KEY_CAPSLOCK    58
#define KEY_LEFTMETA   125
#define KEY_COMPOSE    127
```

**这是全链路第一个"统一命名空间"。** 不管你插的是 USB 键盘还是笔记本内置键盘，
`Esc` 到这里都是 `1`。后面所有层都建立在 keycode 之上。

转换完成后，内核在 `/dev/input/eventN` 上写出 `input_event` 结构：

```c
struct input_event {
    struct timeval time;
    __u16 type;   /* EV_KEY / EV_MSC / EV_SYN */
    __u16 code;   /* type=EV_KEY 时是 keycode */
    __u32 value;  /* 0=抬起  1=按下  2=自动重复 */
};
```

按一次 `Esc` 实际会产生一组事件：

```text
EV_MSC  MSC_SCAN  0x01     ← 原始 scancode，供调试和自定义映射用
EV_KEY  KEY_ESC   1        ← 按下
EV_SYN  SYN_REPORT 0       ← 同步分隔符
EV_MSC  MSC_SCAN  0x01
EV_KEY  KEY_ESC   0        ← 抬起
EV_SYN  SYN_REPORT 0
```

注意 `EV_MSC/MSC_SCAN` 这一条：内核把原始 scancode 也一并暴露出来了。这正是
`setkeycodes` 和 keyd 之类工具能工作的基础——你可以在这一层就改掉映射，对上面所有
层透明。

设备节点可以按物理路径稳定引用：

```console
$ ls /dev/input/by-path/
pci-0000:00:14.0-usb-0:4:1.0-event-kbd     ← 外接 USB 键盘
platform-i8042-serio-0-event-kbd           ← 内置键盘
```

`by-path` 和 `by-id` 这两个目录很重要：第 5 篇讲 udev 热插拔时会用到。

### 观测方法

```console
# 看原始 scancode（Ctrl+D 退出，10 秒无输入自动退出）
$ sudo showkey -s

# 看内核 keycode
$ sudo showkey -k

# 看完整 input_event 流，包括 EV_MSC
$ sudo evtest /dev/input/by-path/platform-i8042-serio-0-event-kbd
```

`showkey` 需要在真实 tty（`Ctrl+Alt+F2`）里运行，因为它要把控制台切到 raw 模式。

## 第 3 层：分叉——控制台、X11 与 Wayland

从这里开始，同一个 keycode 走上三条不同的路。

### 3a. 文本控制台

在 tty 里，内核自己维护一张 keymap，把 keycode 加上修饰键状态翻译成字符或动作
函数。这张表用 `loadkeys` 加载、`dumpkeys` 导出：

```console
$ sudo dumpkeys | head -20
keymaps 0-127
keycode   1 = Escape
keycode  58 = Caps_Lock
```

这条路径完全在内核里，跟 X、跟输入法都无关。它的存在意义是：**当图形层出问题时，
tty 是最后的退路**。第 5 篇讲 keyd 配置写错把键盘锁死时，会用到这一点。

### 3b. X server

X server 通过 evdev 驱动读 `/dev/input/eventN`，然后做一件让所有人困惑过的事：

```text
X keycode = 内核 keycode + 8
```

偏移 8 是历史遗留——X11 协议规定 keycode 取值范围是 8–255，0–7 保留。所以：

| 键 | 内核 keycode | X keycode |
| --- | --- | --- |
| `Esc` | 1 | 9 |
| `` ` `` | 41 | 49 |
| `Caps Lock` | 58 | 66 |
| 左 `Alt` | 56 | 64 |
| 左 `Super` | 125 | 133 |
| `Menu` | 127 | 135 |

记住这张表，第 4 篇读 `.Xmodmap` 时全靠它。当你在配置文件里看到 `keycode 66` 时，
那说的就是 Caps Lock 键。

拿到 X keycode 之后，**XKB** 负责把它翻译成 **keysym**——一个跟"含义"绑定的符号，
比如 `Escape`、`grave`、`Control_L`、`XF86AudioRaiseVolume`。翻译要考虑当前的修饰键
状态和 group（键盘布局组）：同一个 keycode 9，不按 Shift 是 `grave`，按了 Shift 是
`asciitilde`。

X server 最后把 `XKeyEvent` 发给客户端，里面同时带着 keycode 和修饰键状态位。

### 3c. Wayland

Wayland compositor 通过 libinput 直接读 evdev，然后把 keycode 和键盘布局（一份
xkb keymap 的文件描述符）发给客户端，由客户端自己用 **libxkbcommon** 完成
keycode → keysym 的翻译。

差别在于：X11 下翻译发生在 server，`xmodmap` 改的是 server 的全局状态；Wayland 下
翻译发生在客户端，`xmodmap` 完全无效。**这是第 4、5 篇里"为什么最终要下沉到内核层"
的关键论据之一。**

## 第 4 层：窗口管理器

X server 要把事件发给谁？默认是当前有输入焦点的窗口。但窗口管理器可以用
`XGrabKey` 提前注册："`Alt+Return` 这个组合无论焦点在哪，都先给我。"

AwesomeWM 的 `root.keys(globalkeys)` 做的就是这件事。我的配置里有 40 多个这样的
全局键：

```lua
-- config/awesome/rc.lua
modkey = "Mod1"

globalkeys = gears.table.join(
    awful.key({ modkey }, "Return", function() awful.spawn(terminal) end, ...),
    awful.key({ modkey }, "j",      function() awful.client.focus.byidx(1) end, ...),
    awful.key({ modkey }, "space",  function() awful.spawn("rofi -show drun") end, ...),
    ...
)
```

**被 grab 的键不会到达任何应用程序。** 这是最常见的"快捷键失灵"原因之一：你在
Neovim 里绑了 `Alt+j`，但 AwesomeWM 已经把 `Alt+j` 抢走用来切换窗口了，Neovim 永远
收不到它。

第 6 篇会详细讲怎么定位这类冲突，第 7 篇讲全局键应该抢什么、不该抢什么。

## 第 5 层：输入法

事件到达应用程序窗口后，如果这个程序接了输入法框架（通过 `XMODIFIERS=@im=fcitx`、
`GTK_IM_MODULE=fcitx` 等环境变量），按键会**先经过 fcitx5**：

```console
$ env | rg 'IM_MODULE|XMODIFIERS'
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
```

fcitx5 把按键交给当前输入法引擎（我这里是 rime）。引擎有两个选择：

- **吞掉**：比如中文状态下按 `j`，rime 把它当拼音吃进去，应用程序什么也收不到，
  只会通过另一条通道收到一段 preedit 文本。
- **透传**：比如英文状态下按 `j`，rime 不感兴趣，事件继续往下走。

**这一层是整个系列里最容易被忽略、又最容易出问题的一层。** 因为它的行为取决于一个
不可见的状态——当前是中文还是英文。第 12 篇整篇都在讲这件事：

> tmux prefix 配置为 `Alt+b`。在 rime 中文状态下按 `Alt+b` 之后再按 `j`，`j` 会先被
> rime 当作拼音拦截，tmux 收不到命令键。必须再按一次 Shift 切到英文，tmux 绑定才会
> 触发。

这就是"状态确认"成本的典型案例：你必须先知道自己在什么输入状态，才能预测按键会去
哪里。

## 第 6 层：终端模拟器

按键穿过输入法，到达 WezTerm。终端模拟器的工作是把 keysym + 修饰键状态**编码成
字节**，写进 PTY master。

普通字符很直接：`a` → `0x61`。但特殊键没有对应字符，只能用**转义序列**表示：

| 按键 | 发送的字节 |
| --- | --- |
| `Enter` | `0x0d` (CR) |
| `Tab` | `0x09` |
| `Esc` | `0x1b` |
| `Ctrl+c` | `0x03` |
| `↑` | `ESC [ A` = `1b 5b 41` |
| `Alt+b` | `ESC b` = `1b 62` |

这个编码方案是整个系列里最多"坑"的地方，因为它是有损的：

- `Ctrl+i` 和 `Tab` 都编码成 `0x09`，程序无法区分。
- `Ctrl+m` 和 `Enter` 都是 `0x0d`。
- `Ctrl+[` 和 `Esc` 都是 `0x1b`。
- `Ctrl+Shift+h` 根本没有传统编码。
- `Alt+b` 发的是 `ESC b`，跟"先按 Esc 再按 b"字节完全一样——这就是 tmux 需要
  `escape-time` 设置的原因。

现代终端用 **CSI u** 或 **kitty keyboard protocol** 扩展来解决，把修饰键信息完整
编码进转义序列。我的配置两边都开了：

```lua
-- wezterm.lua
enable_csi_u_key_encoding = true
```

```tmux
# .tmux.conf
set -g extended-keys on
set -g extended-keys-format csi-u
```

第 3 篇会把这套编码规则彻底讲清楚。

亲手看一眼字节的办法很简单——在终端里按 `Ctrl+v` 再按目标键：

```console
$ cat -v
^[[A          ← 按了上箭头
^[b           ← 按了 Alt+b
```

## 第 7 层：行规程（termios）

字节写进 PTY master 之后，不会直接到达 shell。中间隔着内核的**行规程（line
discipline）**，由 termios 配置。它在规范模式（canonical mode）下负责：

- **行缓冲**：攒够一整行（遇到 `\n`）才交给读取的程序。
- **回显**：把你输入的字符打回终端。
- **特殊字符处理**：`Ctrl+c`（`0x03`）不是传给程序的数据，而是**产生 `SIGINT`
  信号**；`Ctrl+z` 产生 `SIGTSTP`；`Ctrl+d` 表示 EOF；`Ctrl+s`/`Ctrl+q` 是流控。

```console
$ stty -a | head -3
speed 38400 baud; rows 50; columns 200; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U; eof = ^D;
```

这解释了一个经常被问的问题：**为什么 `Ctrl+c` 在几乎所有终端程序里都不能重新绑定？**
因为它在到达程序之前就被行规程转成信号了。想要拿到原始的 `0x03`，程序必须主动关掉
`ISIG`——这正是 Neovim、tmux、fzf 这类全屏程序启动时做的第一件事：切到 raw 模式，
把行规程的所有加工全部关掉，自己处理每一个字节。

## 第 8 层：终端里的程序

在我的环境里，PTY 另一端是 tmux。tmux 处于一个特殊位置：它既是"终端里的程序"，
又是内层程序的"终端"。

于是同一个按键要被**解码-再编码**两次：

```text
WezTerm ──字节──▶ PTY ──▶ tmux 解码
                              │
                    ┌─────────┴─────────┐
                    │                   │
            匹配 prefix 绑定？    不匹配，重新编码
                    │                   │
                 tmux 自己处理    ──▶ 内层 PTY ──▶ Neovim / zsh
```

tmux 收到 `Alt+b`（`ESC b`）时会查自己的 key table；如果匹配到 `prefix`，就进入
prefix 状态等待下一个键。否则原样转发给内层程序。

最内层的 zsh 和 Neovim 再各自解码一次：

- **zsh** 的 ZLE 维护一棵按键序列前缀树，`bindkey` 就是往这棵树里插节点。
- **Neovim** 用自己的 `termcodes` 解析，然后查 `:map` 表。

至此，一次按键的旅程结束。

## 每一层的观测工具

排查问题时，最有效的办法是**从下往上逐层确认信号还在不在**。这张表是后面所有诊断的
基础：

| 层 | 观测工具 | 看到什么 |
| --- | --- | --- |
| 1 传输 | `lsusb -v`、`usbhid-dump` | HID report descriptor |
| 2 内核 | `sudo showkey -s` | 原始 scancode |
| 2 内核 | `sudo showkey -k` | 内核 keycode |
| 2 内核 | `sudo evtest <dev>` | 完整 input_event 流 |
| 3a 控制台 | `dumpkeys` | 内核 keymap |
| 3b X11 | `xev -event keyboard` | X keycode + keysym + 修饰键状态 |
| 3b X11 | `xmodmap -pke` / `-pm` | 当前 keycode→keysym 表 / modifier 组 |
| 3b Wayland | `wev` | 同上，Wayland 版 |
| 4 窗管 | `Mod+s`（awesome 帮助） | 已注册的全局键 |
| 5 输入法 | `fcitx5-diagnose` | 输入法接入状态 |
| 6 终端 | `cat -v` 或 `Ctrl+v` | 终端实际发出的字节 |
| 6 终端 | `wezterm show-keys --lua` | 终端自己的键位表 |
| 7 行规程 | `stty -a` | 特殊字符与模式 |
| 8 tmux | `tmux list-keys -T prefix` | tmux 键位表 |
| 8 zsh | `bindkey` | ZLE 键位表 |
| 8 Neovim | `:map`、`:verbose map <key>` | 映射及其定义位置 |

## 三个跨层故障

有了这条链路，回头看开头那几个问题，就能定位到具体是哪一层：

**一、`Alt+b` 在中文状态下失效。**
故障点在第 5 层（输入法）。rime 在中文态把后续字母键吞掉了，第 6 层的 tmux prefix
状态机等不到它的命令键。修法可以放在第 5 层（让 rime 别吞）或第 6 层（终端截获后
先切英文再转发）——我最终选了后者，理由见第 12 篇。

**二、`Ctrl+i` 和 `Tab` 在 Neovim 里绑不开。**
故障点在第 6 层（终端编码）。两者被编码成同一个字节 `0x09`，信息在编码时就丢了，
第 8 层的 Neovim 无论如何也恢复不出来。解法是让第 6 层用 CSI u 编码保留信息，
见第 3 篇。

**三、插上外接键盘后 `xmodmap` 失效。**
故障点在第 3b 层。`xmodmap` 的设置是针对当时存在的设备的，X server 在热插拔时会
重新构建键盘映射，把设置冲掉。传统解法是用 udev 规则在热插拔时重跑 `xmodmap`
（我以前就是这么干的，见第 4 篇），更彻底的解法是把映射下沉到第 2 层，见第 5 篇。

## 小结

一次按键要穿过至少 8 个层次，每一层都有自己的命名体系和拦截能力。记住三件事：

1. **内核 keycode 是第一个统一命名空间**，硬件差异到这里就消失了。往下改（第 2 层）
   影响所有上层，往上改（第 4–8 层）只影响那一层及以上。
2. **每一层都能吃掉按键。** 排查"快捷键失灵"的正确姿势是从下往上确认信号还在不在，
   而不是反复改配置试。
3. **终端的字节编码是有损的。** 第 6 层的编码方案决定了第 8 层能绑什么键，这个约束
   无法在上层绕开。

下一篇会把"同一个键在每一层叫什么名字"这件事讲透：
[四层编码：scancode、keycode、keysym 与字符](02-key-encoding.html)。
