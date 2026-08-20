---
title: "Linux 按键体系（八）：终端层 WezTerm"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第八篇。
> 责任链上的第 ④ 层：终端模拟器。它是"图形世界"和"字节世界"的边界，也是整条链上
> 唯一同时看得见 keysym 和字节的地方。

## 一个几乎什么都不抢的终端

WezTerm 配置里最重要的一行是这个：

```lua
-- config/wezterm/wezterm.lua
disable_default_key_bindings = true,
```

**把 WezTerm 自带的全部默认键位关掉。** 默认配置里有几十个绑定：调字号、切 tab、
分屏、滚动、搜索、复制模式……全部作废，从零开始。

理由回到 [第六篇](06-who-gets-the-key.html) 的"越低层越克制"：终端在责任链的第 ④
层，它抢走的键，下面的 tmux、zsh、Neovim 就永远拿不到。而这三者的键位传统（readline、
vim、tmux prefix）远比终端自己的默认键位重要。

关掉之后，我只加回了 11 个绑定。全部列在这里：

```lua
-- config/wezterm/keybinds.lua
M.default_keybinds = {
	{ key = "c", mods = "CTRL|SHIFT", action = act({ CopyTo = "Clipboard" }) },
	{ key = "v", mods = "CTRL|SHIFT", action = act({ PasteFrom = "Clipboard" }) },
}

M.tmux_keybinds = {
	{ key = "b", mods = "ALT",        action = wezterm.action_callback(switch_rime_to_en_and_send_tmux_prefix) },
	{ key = "t", mods = "CTRL|SHIFT", action = act({ SpawnTab = "CurrentPaneDomain" }) },
	{ key = "1", mods = "ALT",        action = act({ ActivateTab = 0 }) },
	{ key = "2", mods = "ALT",        action = act({ ActivateTab = 1 }) },
	{ key = "3", mods = "ALT",        action = act({ ActivateTab = 2 }) },
	{ key = "4", mods = "ALT",        action = act({ ActivateTab = 3 }) },
	{ key = "5", mods = "ALT",        action = act({ ActivateTab = 4 }) },
	{ key = "6", mods = "ALT",        action = act({ ActivateTab = 5 }) },
}
```

分成三类：

- **`Ctrl+Shift+c/v/t`** —— 剪贴板和新 tab。
- **`Alt+1`–`Alt+6`** —— 切 tab。
- **`Alt+b`** —— 一个特殊的回调，本文后半段详细讲。

**为什么不抢分屏？** 因为 tmux 已经做了这件事。终端和 tmux 都能分屏，但两套 pane
互不知情：终端分出来的 pane 里各跑一个 shell，tmux 完全看不见。同时用两套，你就要
一直回答"我现在这个格子是谁分的"——典型的状态确认成本。所以**只保留一套，并且选
能跨 ssh 保持的那套**（tmux 的会话在断线后还在）。

WezTerm 的 tab 保留了，因为它跟 tmux 的 window 用途不同：tab 用来分隔"不同的机器 /
不同的 tmux 会话"，tmux window 用来分隔一个会话内的任务。

## 为什么是 `Ctrl+Shift+*`

这三个绑定用 `Ctrl+Shift` 而不是 `Ctrl` 有硬性原因，回到
[第三篇](03-terminal-keys.html)：

- `Ctrl+c` 必须原样传给程序（`0x03` → 行规程转 `SIGINT`）。终端抢了它，你就无法
  中断任何程序。
- `Ctrl+v` 在 zsh 和 Neovim 里是"下一个键按原样插入"，是排查按键问题的主力工具
  （[第六篇](06-who-gets-the-key.html) 的诊断流程里用了三次）。
- **传统编码下 `Ctrl+Shift+字母` 根本不存在**（`Ctrl+a` 和 `Ctrl+Shift+a` 都是
  `0x01`）。所以终端拿走它，零成本——本来就没有任何程序能收到它。

这是"选那些不可能被占用的组合"这条原则最干净的应用。

## leader：一个预留的命名空间

```lua
leader = { key = "Space", mods = "CTRL|SHIFT" },
```

`Ctrl+Shift+Space` 作为 leader。同样的理由：传统编码下这个组合不存在，开了 CSI u
之后才能用，所以不可能跟任何既有约定冲突。

**诚实地说：目前没有任何绑定实际用到这个 leader。** 它是预留的命名空间——一旦将来
需要给 WezTerm 加一批终端专属操作（比如工作区管理、多路复用域切换），有一整个前缀
可用，不需要再去跟下面几层抢键。

这算不算过度设计？我认为不算，理由是**预留一个前缀的成本是零**（它不占用任何实际
可用的组合），而将来临时找键的成本很高（要重新审视整条责任链）。

## CSI u：终端这一侧的开关

```lua
enable_csi_u_key_encoding = true,
```

[第三篇](03-terminal-keys.html) 讲过，这让 `Ctrl+i` / `Tab`、`Ctrl+Shift+*` 这类
组合能被无歧义编码。但也讲过它必须**逐层贯通**：

```text
WezTerm (enable_csi_u_key_encoding)
   ↓
tmux (extended-keys on + extended-keys-format csi-u)
   ↓
Neovim (原生支持)
```

三处都开了，本地环境下扩展键位可用。但 ssh 到没配置的远端时会静默降级——所以我的
Neovim 核心键位仍然只用传统编码能表达的组合，见 [第十一篇](11-neovim.html)。

验证终端这一侧：

```console
$ wezterm show-keys --lua | rg "key = 'b'|ALT|user-defined"
```

## 输入法集成

```lua
use_ime = true,
ime_preedit_rendering = "Builtin",
use_dead_keys = false,
```

三行都跟 [第十二篇](12-rime-auto-switch.html) 相关：

- **`use_ime = true`** —— 走 XIM/IBus 协议接入 fcitx5。关掉的话终端里根本无法输入
  中文。
- **`ime_preedit_rendering = "Builtin"`** —— 拼音预编辑串由 WezTerm 自己在光标位置
  渲染，而不是让 fcitx5 弹一个独立的浮动窗口。**这是"保护注意力"的一个具体选择**：
  预编辑文本出现在你眼睛已经在看的位置，视线不需要移动。
- **`use_dead_keys = false`** —— 关掉死键（欧洲键盘输入重音符号用的两步输入）。
  中文环境用不到，而它会让某些按键出现"按了没反应，要再按一下"的行为——正是要避免的
  那种状态。

## `action_callback` + `SendKey`：正确的转发模式

`Alt+b` 那条绑定是整个配置里最有意思的一条：

```lua
local ensure_rime_ascii_mode = [[
if ! busctl --user call org.fcitx.Fcitx5 /rime org.fcitx.Fcitx.Rime1 IsAsciiMode 2>/dev/null | grep -q 'b true'; then
	busctl --user call org.fcitx.Fcitx5 /rime org.fcitx.Fcitx.Rime1 SetAsciiMode b true >/dev/null 2>&1 || true
fi
]]

local function switch_rime_to_en_and_send_tmux_prefix(window, pane)
	wezterm.run_child_process({ "sh", "-lc", ensure_rime_ascii_mode })
	window:perform_action(act.SendKey({ key = "b", mods = "ALT" }), pane)
end

M.tmux_keybinds = {
	{ key = "b", mods = "ALT", action = wezterm.action_callback(switch_rime_to_en_and_send_tmux_prefix) },
}
```

它解决的问题在 [第六篇](06-who-gets-the-key.html) 的案例一：中文状态下按 `Alt+b`
进 tmux prefix，后续的 `j/k/h/l` 会被 rime 当拼音吃掉。

做法是：**在终端层截获 `Alt+b`，先把 rime 切成英文，再把这个键原样转发下去。**
tmux 那边完全不知道发生过什么，prefix 逻辑一行都不用改。

### 为什么是 `SendKey` 而不是 `xdotool`

这是本文最重要的一个技术点。合成按键有两条路：

```text
方案 A：xdotool key alt+b
   → 通过 X11 的 XTEST 扩展注入事件
   → 从责任链的 ② 层重新走一遍
   → AwesomeWM 也会看到！

方案 B：window:perform_action(act.SendKey(...), pane)
   → WezTerm 直接把编码后的字节写进那个 pane 的 PTY
   → 完全不经过 X
```

[第六篇](06-who-gets-the-key.html) 的案例三就是方案 A 的翻车现场：

> 如果在这里用 `xdotool --clearmodifiers` 模拟按键，会释放/恢复 Alt，干扰
> Awesome WM 看到的修饰键状态，导致 `Alt+b` 后的 `j` 有时变成 WM 的 `Alt+j`
> 快捷键。

`SendKey` 没有这个问题，因为它作用在**责任链的下游**，不会倒流回上层。

**推广成一条通用规则：需要程序化地"发送一个按键"时，优先找目标程序自己的注入接口
（WezTerm 的 `SendKey`、tmux 的 `send-keys`、Neovim 的 `nvim_feedkeys`），最后才
考虑 X11 层的合成事件。** 后者会重走整条责任链，副作用范围远超预期。

### 幂等与去抖

注意那段 shell 先 `IsAsciiMode` 查询，只在**不是**英文时才 `SetAsciiMode`：

```sh
if ! busctl ... IsAsciiMode 2>/dev/null | grep -q 'b true'; then
  busctl ... SetAsciiMode b true >/dev/null 2>&1 || true
fi
```

为什么不无条件设置？因为 fcitx5 每次状态变化会弹一个提示。无条件调用的话，你每次按
tmux prefix 都会看到一次输入法提示闪过——**一个为了减少干扰而做的功能，自己变成了
干扰源**。

这是个很小的细节，但它恰好体现了系列主线：**判断标准不是"功能有没有实现"，而是
"用户的注意力有没有被打扰"。**

## 光标：不闪烁

```lua
animation_fps = 1,
cursor_blink_ease_in = "Constant",
cursor_blink_ease_out = "Constant",
cursor_blink_rate = 0,
force_reverse_video_cursor = true,
```

`cursor_blink_rate = 0` 关掉光标闪烁。

这不是按键配置，但它属于同一个主题。**闪烁是周期性的运动，而人的周边视觉对运动
高度敏感**——这是进化留下的本能。一个每 500ms 闪一次的光标，会持续不断地把注意力
往回拉。

`force_reverse_video_cursor` 让光标用反色显示，这样无论什么配色方案下都清晰可见。
**不闪烁 + 高对比**，是"要看的时候一眼能找到，不看的时候不打扰"的组合。

`animation_fps = 1` 是同一个思路：把所有动画降到几乎静止。

## 鼠标绑定

```lua
M.mouse_bindings = {
	{
		event = { Up = { streak = 1, button = "Left" } },
		mods = "NONE",
		action = act({ CompleteSelection = "PrimarySelection" }),
	},
	{
		event = { Up = { streak = 1, button = "Right" } },
		mods = "NONE",
		action = act({ CompleteSelection = "Clipboard" }),
	},
	{
		event = { Up = { streak = 1, button = "Left" } },
		mods = "CTRL",
		action = "OpenLinkAtMouseCursor",
	},
}
```

左键选中即进 primary selection（中键粘贴，X11 传统），右键选中进系统剪贴板，
`Ctrl+左键` 打开链接。

**三个绑定，没有一个需要按修饰键就能完成最常用的操作。** 鼠标操作的设计原则跟键盘
一样：最高频的路径不应该有前置条件。

## ssh domain 自动生成

一个跟键位无关但值得一提的设计：

```lua
local function create_ssh_domain_from_ssh_config(ssh_domains)
	if ssh_domains == nil then
		ssh_domains = {}
	end
	for host, config in pairs(wezterm.enumerate_ssh_hosts()) do
		table.insert(ssh_domains, {
			name = host,
			remote_address = config.hostname .. ":" .. config.port,
			username = config.user,
			multiplexing = "None",
			assume_shell = "Posix",
		})
	end
	return { ssh_domains = ssh_domains }
end
```

从 `~/.ssh/config` 自动生成 WezTerm 的 ssh domain 列表。**单一数据源**——加一台机器
只改 `~/.ssh/config`，不用同步两份配置。

这属于系列主线的第三条标准"可长期维护"：任何需要在两个地方同步的配置，早晚会不同步。

## 小结

1. **`disable_default_key_bindings = true` 是起点**，终端在责任链第 ④ 层，抢走的键
   下面永远拿不到。最终只保留 11 个绑定。
2. **`Ctrl+Shift+*` 是终端层的天然领地**：传统编码下不存在，抢它零成本；而
   `Ctrl+c`/`Ctrl+v` 必须留给下游。
3. **leader 可以预留不用**，成本为零，将来省一次全链路审视。
4. **`SendKey` 而不是 `xdotool`**：程序化发键要用目标程序自己的注入接口，X11 合成
   事件会从 ② 层重走整条责任链。
5. **先查询再设置**，避免无条件调用触发输入法提示——功能不该自己变成干扰源。
6. **关掉光标闪烁和动画**：周边视觉对运动敏感，静止的界面才不抢注意力。

下一篇看复用层，以及那些真正决定体验的"隐形"设置：
[复用层：tmux](09-tmux.html)。
