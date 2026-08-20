---
title: "Linux 按键体系 05：到内核层重塑键盘"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第五篇。
> [上一篇](04-xmodmap.html) 把 `xmodmap` 讲透了，也数清了它的天花板：热插拔失效、
> Wayland 无效、tty 无效、没有 tap/hold、没有分层。本篇把映射下沉到内核层。

## 为什么要下沉一层

回顾 [第一篇](01-key-journey.html) 的链路图，`xmodmap` 工作在第 3b 层（X server）。
在它下面还有第 2 层（内核 input 子系统），而第 2 层有一个决定性的优势：

```text
                    ┌─ tty（内核 keymap）
第 2 层 内核 input ──┼─ X server ──▶ xmodmap 在这里
                    └─ Wayland compositor
```

**所有上层都从第 2 层取数据。** 在第 2 层改掉映射，tty、X、Wayland、以及通过 ssh
进来的会话，看到的都是改过之后的结果。不需要三份配置，不需要 udev 补丁。

更重要的是，第 2 层能做上层做不到的事：**tap/hold** 和**分层**。这两个特性能在不增加
组合键复杂度的前提下扩展可用键位——直接对应系列主线里"保护注意力"那一条。

## keyd 的工作原理

[keyd](https://github.com/rvaiya/keyd) 是一个 systemd 服务，做三件事：

```text
   物理键盘
   /dev/input/eventN
        │
        │ ① EVIOCGRAB —— 独占抓取，上层看不到原始事件
        ▼
      keyd 守护进程
        │
        │ ② 按配置重新映射（含 tap/hold、分层、宏）
        ▼
   ③ 通过 uinput 创建虚拟键盘设备
   /dev/input/eventM  ("keyd virtual keyboard")
        │
        ▼
   X server / Wayland / tty 看到的是这个虚拟设备
```

三个要点：

1. **`EVIOCGRAB` 独占**：keyd 抓取真实设备后，其他程序（包括 X server）就读不到它
   的原始事件了。
2. **uinput 造一个虚拟键盘**：keyd 把处理后的事件从这个虚拟设备发出去。
3. **对上层完全透明**：X server 只看到一个普通键盘，不知道中间有人动过手脚。

因为处理发生在 evdev 层，keyd 天然支持热插拔——它监听设备添加事件，按 `[ids]` 段
自动匹配并接管新键盘。[第四篇](04-xmodmap.html) 里那套 udev 规则加 `sleep 0.2` 的
补丁，在这里整个不需要了。

## 安装与最小配置

```console
$ sudo apt install keyd          # Debian 13+；旧版本需要从源码编译
$ sudo systemctl enable --now keyd
```

配置在 `/etc/keyd/default.conf`。最小可用配置：

```ini
[ids]
*

[main]
capslock = overload(control, esc)
```

三行做了一件事：**轻点 Caps Lock 是 Esc，按住是 Ctrl**。

这一条配置就抵得上 [第四篇](04-xmodmap.html) 里整个三键轮换加 udev 那一套，而且
tty 里也生效。

改完重载：

```console
$ sudo keyd reload
```

## 配置语法

### `[ids]` —— 匹配哪些设备

```ini
[ids]
*                    # 所有键盘
-0123:4567           # 但排除这一个
```

或者只针对特定设备：

```ini
[ids]
0f39:0611            # Poker II（vendor:product）
```

这个 `0f39:0611` 正是我 udev 规则里那个：

```
ATTR{idVendor}=="0f39", ATTR{idProduct}=="0611", ...
```

查设备 ID：

```console
$ sudo keyd monitor
# 或
$ lsusb
```

**一份配置文件可以有多个 `[ids]` 段吗？** 不能——每个文件一个 `[ids]`。多把键盘要用
多个配置文件，`/etc/keyd/` 下的每个 `.conf` 独立匹配。这正好解决了
[第四篇](04-xmodmap.html) 里 `.pokerXmodmap` / `.tpXmodmap` 那个问题：

```
/etc/keyd/thinkpad.conf   →  [ids] 里写内置键盘的 id
/etc/keyd/poker.conf      →  [ids] 0f39:0611
```

插上 Poker II，keyd 自动用 `poker.conf` 接管它；拔掉，什么也不用做。

### `[main]` —— 默认层

```ini
[main]
capslock = esc
esc = capslock
leftalt = leftcontrol
leftmeta = leftalt
```

左边是键名（用内核键名的小写形式，`keyd list-keys` 可以列全），右边是动作。

### 层

```ini
[main]
capslock = overload(nav, esc)

[nav]
h = left
j = down
k = up
l = right
```

按住 Caps 时进入 `nav` 层，`hjkl` 变方向键；轻点 Caps 还是 Esc。

层还可以带修饰键语义：

```ini
[nav:C]              # 这个层同时表现得像按住了 Ctrl
```

## 核心能力一：tap/hold

`overload(layer, action)` 是 keyd 最有价值的原语：

```ini
capslock = overload(control, esc)
```

- 轻点（按下后很快抬起，且期间没按别的键）→ 发送 `esc`
- 按住（或期间按了别的键）→ 表现为 `control`

**为什么这个特性重要？** 它让一个物理键承担两个职责，而且这两个职责不会互相干扰。
Caps Lock 位置极好（主键区左侧，小指自然落点），但它本身的功能几乎没人用。把它变成
"Esc + Ctrl"，等于凭空多出一个黄金位置的修饰键，同时把 Esc 从键盘左上角搬到了手边。

对 Vim 用户来说这是双重收益：退出插入模式（极高频）和 Ctrl 组合（极高频）都不再需要
移动手掌。

需要注意的是**误触发**。快速打字时，如果你在按住 Caps 的同时按了另一个键，keyd 会
判定为 hold。这在 Caps 上几乎不出问题，但如果你想把 `overload` 放到空格或字母键上，
就要调超时：

```ini
[global]
overload_tap_timeout = 200
```

具体有哪些超时选项、语义如何，跟版本有关，装好后 `man keyd` 是唯一可靠的来源。

## 核心能力二：分层

分层的价值在于**不增加组合键的宽度**。

对比两种方案做同一件事（Home/End/PageUp/PageDown）：

```text
方案 A（组合键）：Ctrl+a / Ctrl+e / Alt+v / Ctrl+v
                  → 四个分散的组合，跟终端程序的既有键位冲突

方案 B（分层）：  按住 Caps 时 u/i/o/p
                  → 一个空间位置，零冲突，手不离开主键区
```

方案 B 更容易形成肌肉记忆，因为它是**空间连续**的——`hjkl` 是方向，`ui op` 是翻页，
它们在物理上挨在一起。方案 A 的四个组合彼此没有空间关系，只能死记。

这是系列主线"保护注意力"的一个具体落点：**能用空间关系编码的，就别用符号记忆编码。**

## 迁移：把我的 xmodmap 配置搬过来

[第四篇](04-xmodmap.html) 那份 `~/.Xmodmap` 干了三件事：三键轮换、把物理 Alt 变成
Ctrl、把物理 Windows 键变成 mod1。搬到 keyd：

```ini
# /etc/keyd/default.conf
[ids]
*

[main]
# 三键轮换：Caps → Esc，Esc → `，` → Caps
capslock = esc
esc = grave
grave = capslock

# 交换 Alt / Super：拇指位置给 Ctrl，Windows 键给 WM
leftalt  = leftcontrol
leftmeta = leftalt

# 禁用 Menu 键
compose = noop
```

对照一下两边的规模：

| | xmodmap 方案 | keyd 方案 |
| --- | --- | --- |
| 配置文件 | `.Xmodmap` + `.pokerXmodmap` + `.tpXmodmap` | 1 个（或按键盘拆成 2 个） |
| 热插拔支撑 | udev 规则 + 2 个 shell 脚本 + `sleep 0.2` | 无需 |
| 登录时挂载 | `.xinitrc` 里判断设备 | 无需 |
| tty 生效 | 否 | 是 |
| Wayland 生效 | 否 | 是 |
| 幂等 | 否 | 是 |

而且既然已经在这一层了，顺手就能加上 xmodmap 永远做不到的：

```ini
[main]
capslock = overload(nav, esc)

[nav]
h = left
j = down
k = up
l = right
u = pageup
d = pagedown
```

## 安全：不要把自己锁在门外

**这是使用 keyd 唯一需要认真对待的风险。** 它独占了键盘设备，配置写错可能让整个
键盘失去响应，而你连打开编辑器改回来都做不到。

四道保险，按可靠性排序：

**一、先在另一台机器上 ssh 进来。** 最可靠。改配置之前先开好一个 ssh 会话，出问题
直接 `systemctl stop keyd`。

**二、用 `keyd -m` 或独立进程测试。** 不要直接改 `/etc/keyd/default.conf` 然后
`reload`，先用临时配置验证。

**三、留一把不受 keyd 管的键盘。** `[ids]` 里排除某个设备（比如一把便宜的 USB 小
键盘），它就永远是原样，可以用来救急。

**四、keyd 内置的应急组合键。** keyd 文档给出了一个"同时按下 backspace + escape +
enter"来终止守护进程的应急序列。不同版本可能不同，装好后先 `man keyd` 确认当前版本
的行为，别等到需要时才查。

另外记住 [第一篇](01-key-journey.html) 提到的那条退路：**tty 走的是内核 keymap**，
但 keyd 在更底层，所以 tty 也会受影响。真正独立于 keyd 的退路只有 ssh 和物理上换一把
被排除的键盘。

## 调试

```console
# 实时看 keyd 收到和发出的事件，同时显示设备 id
$ sudo keyd monitor

# 列出所有合法键名
$ keyd list-keys

# 检查配置语法并重载
$ sudo keyd reload
$ sudo systemctl status keyd

# 看 keyd 接管了哪些设备
$ sudo keyd monitor       # 开头会列出匹配到的设备

# 确认虚拟设备存在
$ rg -A2 'keyd' /proc/bus/input/devices
```

配合 [第一篇](01-key-journey.html) 的观测表逐层验证：

```console
$ sudo showkey -k                    # 内核 keycode（keyd 之后的结果）
$ xev -event keyboard                # X keycode + keysym
```

注意 `showkey -k` 看到的已经是 keyd 处理后的结果，因为 keyd 在更底层。想看原始事件
要用 `evtest` 直接读物理设备节点。

## 其他方案

keyd 不是唯一选择，各有取舍：

| 方案 | 层次 | 特点 |
| --- | --- | --- |
| **键盘固件**（QMK / VIA / ZMK） | 第 0 层 | 最彻底，换电脑也跟着走；但要键盘支持刷固件 |
| **keyd** | 第 2 层 | 配置简单，systemd 服务，够用 |
| **kanata** / **kmonad** | 第 2 层 | 功能更强（组合键、复杂状态机），配置复杂度也更高 |
| **interception-tools** | 第 2 层 | 管道式插件架构，灵活但要自己拼 |
| **XKB** | 第 3b 层 | X/Wayland 都用，但自定义规则门槛高 |
| **xmodmap** | 第 3b 层 | 简单，但限制多（见第四篇） |

如果你的键盘支持 QMK，**优先考虑固件层**——那是唯一一个"跨操作系统、跨机器都一致"的
方案。我的 Poker II 是可编程的，把三键轮换烧进固件，接到任何机器上都对。

## 什么该放 keyd，什么不该

一条实践原则：**keyd 里只放"键的身份"，不放"功能"。**

| 适合放 keyd | 不适合放 keyd |
| --- | --- |
| Caps → Ctrl/Esc | 启动终端 |
| 交换 Alt / Super | 切换窗口 |
| 方向键分层 | 复制粘贴 |
| 修复厂商乱定义的功能键 | 任何跟具体程序相关的操作 |
| 禁用误触的键（Menu、Insert） | 输入法切换 |

原因是**作用域**。keyd 的改动对所有程序生效，无法关闭。你把"启动终端"绑到 keyd 上，
那这个键在任何程序里都被吃掉了——包括你临时需要它原本功能的场合。这类"有语义的操作"
应该放在窗口管理器（[第七篇](07-awesome.html)）或应用层，因为那些层知道当前上下文，
可以按需让路。

这也呼应 [第二篇](02-key-encoding.html) 的决策表：**低层负责"是什么键"，高层负责
"做什么事"。**

## 小结

1. **keyd 工作在第 2 层**（`EVIOCGRAB` 独占 + uinput 虚拟设备），对 tty、X、Wayland
   一视同仁，并且天然支持热插拔。
2. **`overload(layer, action)` 是最有价值的原语**：一个物理键同时是 Esc 和 Ctrl，
   把两个高频操作都搬到手边。
3. **分层用空间关系代替符号记忆**，比堆组合键更容易形成肌肉记忆，也更不容易冲突。
4. **`[ids]` 按设备匹配**，一举解决多键盘问题——不再需要 udev 规则和逆操作脚本。
5. **务必先准备好退路**（ssh 会话最可靠），keyd 配错能让键盘完全失去响应。
6. **keyd 只放"键的身份"，不放"功能"**；有语义的操作交给知道上下文的上层。

到这里，"键是什么"已经确定了。下一篇进入本系列的核心问题之二——
**同一个键，在六层环境里到底被谁拿走了**：
[谁先拿到这个键：捕获顺序与冲突诊断](06-who-gets-the-key.html)。
