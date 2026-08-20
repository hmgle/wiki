---
title: "Linux 按键体系 12：中英文自动切换"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第十二篇，回答系列的第三个核心
> 问题：**zsh / wezterm / nvim 三处"自动切换 rime 中英文"是如何实现的，底层原理
> 是什么？**
>
> 这也是整个系列里"减少状态确认"这条主线最集中的一次实践。

## 问题：一个看不见的状态机

[第六篇](06-who-gets-the-key.html) 的责任链里，输入法在第 ③ 层——窗口管理器之后，
终端之前。它的行为取决于一个二值状态：

```text
中文态：字母键被 rime 吃掉，变成拼音候选，应用程序收不到
英文态：字母键透传，应用程序正常收到
```

**这个状态是全局的、隐式的、而且跟你正在做的事情完全无关。** 于是每次要按快捷键
之前，你必须先回答"我现在是中文还是英文"。答错的代价是：

| 场景 | 中文态下按键的后果 |
| --- | --- |
| tmux prefix 后按 `j` | 弹出拼音候选，pane 没切换 |
| Neovim 普通模式按 `dd` | 弹出拼音候选，行没删掉 |
| Neovim 普通模式按 `:` | 输入了全角 `：` |
| zsh 里敲命令 | 满屏拼音 |

而且这个状态**没有可靠的视觉锚点**。fcitx5 的状态提示是一闪而过的浮动窗口，
托盘图标要抬眼去看，终端里没有任何指示。

## 两个方向

面对这个问题有两条路：

**方向 A：让状态更可见。** 加状态栏指示器、改光标颜色、让 fcitx5 常驻显示。

**方向 B：让状态自动回到正确值。** 在能确定"接下来一定是英文"的时刻，自动切回英文。

我选了 B，理由是**方向 A 治标**：它把"确认状态"这个动作变便宜了，但没有消除它。
你还是要看一眼、判断、必要时切换——每次都是一次微小的注意力转移。

方向 B 的目标是让"当前是什么输入状态"这个问题**根本不需要被问出来**。

关键洞察是：**在很多时刻，正确的输入状态是可以推断出来的**：

| 时刻 | 接下来几乎一定是 |
| --- | --- |
| Neovim 离开插入模式 | 英文（普通模式命令） |
| Neovim 离开命令行 | 英文 |
| zsh 新 prompt 出现且命令行为空 | 英文（命令名） |
| 按下 tmux prefix | 英文（tmux 命令键） |

这四个时刻，就是三处实现的全部触发点。

## 状态模型：fcitx5 的 active 与 rime 的 ascii_mode

要动手之前必须搞清楚一件事：**这里有两个不同的状态，很容易搞混。**

```text
┌─ fcitx5 层 ──────────────────────────────┐
│  输入法 active / inactive                 │
│  (fcitx5-remote -c 关掉的是这个)          │
│                                           │
│  ┌─ rime 引擎 ────────────────────────┐  │
│  │  ascii_mode = true / false          │  │
│  │  (rime 自己的中英文开关)             │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
```

- **fcitx5 的 active**：整个输入法引擎是否参与处理。inactive 时按键直接透传。
- **rime 的 `ascii_mode`**：rime 引擎内部的中英文模式。`ascii_mode = true` 时
  rime 虽然在工作，但直接输出 ASCII。

`fcitx5-remote -c` 操作的是外层，而我的配置只用 rime 一个输入法方案，真正需要控制的
是内层。二者语义不同，实测下来外层的可靠性不够——记在
`config/wezterm/docs/rime-tmux-prefix.md` 里：

> `fcitx5-remote -c` 的含义是让 fcitx 输入法 inactive。它对当前 Rime 方案的
> `ascii_mode` 不够可靠，实际测试中执行前后状态可能仍然不能让 prefix 后的字母
> 绕过 Rime。

## 接口：fcitx5-rime 的 D-Bus 方法

fcitx5-rime 暴露了两个精确的 D-Bus 方法：

```console
$ busctl --user introspect org.fcitx.Fcitx5 /rime
NAME                              TYPE      SIGNATURE RESULT/VALUE
org.fcitx.Fcitx.Rime1.IsAsciiMode  method   -         b
org.fcitx.Fcitx.Rime1.SetAsciiMode method   b         -
```

用法：

```console
# 查询当前是否英文模式
$ busctl --user call org.fcitx.Fcitx5 /rime org.fcitx.Fcitx.Rime1 IsAsciiMode
b true

# 设置为英文模式
$ busctl --user call org.fcitx.Fcitx5 /rime org.fcitx.Fcitx.Rime1 SetAsciiMode b true
```

**这是整套机制的地基**，三处实现全部建立在这两个调用上。

## 幂等：所有设计的前提

`SetAsciiMode b true` 是**设置目标态**，不是 toggle。这一点至关重要：

```text
toggle 语义：  调用 N 次的结果取决于 N 的奇偶性
              → 并发、重复、乱序调用全部会出错
              → 必须精确控制调用次数

设置语义：    调用 N 次的结果跟调用 1 次相同
              → 并发、重复、乱序调用全都安全
              → 不需要精确控制
```

**这直接决定了整套架构可以是"尽力而为"的。** 后面会看到 tmux 多 client 归因存在
毫秒级竞态——如果动作不幂等，就必须用锁或事务来消除竞态；因为幂等，误判的代价只是
"多设了一次已经是那个值的状态"，无害。

**一条通用经验：设计自动化时，优先把动作做成幂等的设置，而不是 toggle。** 这能省掉
整个类别的同步问题。

同理，rime 那条冷僻快捷键也是 `set_option` 而不是 `toggle`：

```yaml
# ~/.local/share/fcitx5/rime/default.custom.yaml
patch:
  key_binder/bindings/+:
    # 使用极其冷僻的组合键强制设置为英文模式
    - { when: always, accept: "Control+Alt+Shift+F12", set_option: ascii_mode }
```

`Control+Alt+Shift+F12` 选得极冷僻，就是为了不跟任何东西冲突。它是 `xdotool` fallback
路径专用的（下面会讲）。

## 实现一：zsh ZLE

最简单的一处，也是逻辑最清晰的：

```zsh
# home/.zshrc
__rime_ensure_ascii_mode() {
  [[ -n "${DBUS_SESSION_BUS_ADDRESS:-}" ]] || return 0
  [[ -n "${DISPLAY:-}${WAYLAND_DISPLAY:-}" ]] || return 0
  (( $+commands[busctl] )) || return 0

  busctl --user call org.fcitx.Fcitx5 /rime org.fcitx.Fcitx.Rime1 IsAsciiMode 2>/dev/null \
    | command grep -q 'b true' \
    || busctl --user call org.fcitx.Fcitx5 /rime org.fcitx.Fcitx.Rime1 SetAsciiMode b true >/dev/null 2>&1
}

__rime_ascii_mode_on_empty_zle_line() {
  [[ -o interactive ]] || return 0
  [[ -z "${BUFFER:-}" && -z "${LBUFFER:-}" && -z "${RBUFFER:-}" ]] || return 0

  __rime_ensure_ascii_mode
}

if [[ -o interactive ]]; then
  autoload -Uz add-zle-hook-widget
  add-zle-hook-widget line-init __rime_ascii_mode_on_empty_zle_line
fi
```

**触发时机**：`line-init`，也就是每次新命令行开始编辑时。

**三个前置守卫**：没有 D-Bus、没有图形会话、没有 `busctl` 就直接返回。这让同一份
`.zshrc` 在 ssh、tty、容器里都能安全加载。

**先查后设**：只在当前不是英文时才调 `SetAsciiMode`。原因见
[第八篇](08-wezterm.html)——无条件调用会让 fcitx5 每次都弹状态提示，功能自己变成了
干扰源。

**刻意的边界**：只在 `BUFFER`/`LBUFFER`/`RBUFFER` 全空时才切换。

> 这个方案的刻意边界是：它只处理 zsh 新 prompt 出现时的空命令行。如果已经开始
> 编辑命令，例如光标在 `echo ` 后面准备输入中文，`BUFFER` 不为空，不会自动切换。

**为什么放 ZLE 而不是终端层**（[第十篇](10-zsh-zle.html) 展开过）：

> ZLE 只在 shell 正在编辑命令行时运行，因此不会影响 `codex`、`nvim`、`fzf`
> 等子进程自己的输入界面。

**已知局限**（写在文档里，很重要）：

> 如果用户在空 prompt 出现以后又手动切回中文，zsh 无法可靠收到这个输入法状态变化；
> 下一次新 prompt 出现时会再次自动切回英文。

这是个诚实的取舍：zsh 没有订阅输入法状态变化的能力，所以它只能在自己的事件点上
"重申"目标状态。

## 实现二：WezTerm 截获 tmux prefix

场景是 [第六篇](06-who-gets-the-key.html) 的案例一：

> tmux prefix 配置为 `Alt+b`。在 Rime 中文状态下，按 `Alt+b` 以后再按 `j/k/h/l`
> 等 tmux prefix 绑定，普通字母键会先被 Rime 当作拼音输入拦截，tmux 收不到命令键。

注意问题的**结构**：`Alt+b` 本身带修饰键，rime 不感兴趣，透传了；但它开启的
**后续状态**（tmux 等待命令键）里，用户要按的是裸字母键——而那些会被 rime 吃掉。

**两个状态机在物理上共享按键序列，但互不知情。**

解法：在终端层截获 prefix 键，切完输入法再转发。

```lua
-- config/wezterm/keybinds.lua
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

**tmux 那边一行都不用改**：

```tmux
set -g prefix M-b
unbind C-b
bind M-b send-prefix
```

这是这个方案最大的优点——**它在责任链上游"预处理"，下游完全无感**。tmux 的
prefix 逻辑保持原生，时序和容错都是 tmux 自己的实现，没有引入新的状态机。

**为什么用 `SendKey` 而不是 `xdotool`**：[第八篇](08-wezterm.html) 详细讲过，
X11 合成事件会从责任链的 ② 层重走一遍，可能被 AwesomeWM 截获。

**这个方案的边界**（也写在文档里）：

> 如果换终端模拟器，这套 WezTerm key callback 方案不会自动迁移。需要在新终端
> 或窗口管理器层做等价逻辑。

## 实现三：Neovim

最复杂的一处，因为它要处理的场景最多。

### 触发事件矩阵

```lua
-- lua/options.lua
local group = vim.api.nvim_create_augroup('fcitx', { clear = true })

vim.api.nvim_create_autocmd('InsertLeave',   { group = group, pattern = '*', callback = switch_to_en_for_client })
vim.api.nvim_create_autocmd('CmdlineLeave',  { group = group, pattern = '*', callback = function() vim.schedule(switch_to_en_for_client) end })
vim.api.nvim_create_autocmd('TermLeave',     { group = group, pattern = '*', callback = switch_to_en_for_client })
```

| 事件 | 场景 |
| --- | --- |
| `InsertLeave` | 插入模式输入中文后按 Esc |
| `CmdlineLeave` | 命令行里输入中文（如 `/搜索`）后回到普通模式 |
| `TermLeave` | 从终端 buffer（toggleterm、Aider）回到普通模式 |
| `FocusGained` | 在别的程序里打完中文切回来 |
| `<Esc>` 映射 | 兜底 |
| 全角标点映射 | 兜底 |

`FocusGained` 那条有一个精心设计的条件：

```lua
-- Regaining focus after typing Chinese in another application. Skip
-- text-input modes (insert/replace/cmdline/terminal/select) so a quick
-- app switch does not interrupt ongoing Chinese input there.
vim.api.nvim_create_autocmd('FocusGained', {
  group = group,
  pattern = '*',
  callback = function()
    local mode = vim.api.nvim_get_mode().mode
    if not mode:find '^[iRctsS\19]' then
      switch_to_en_for_client()
    end
  end,
})
```

**只在非文本输入模式下才切。** 如果你正在插入模式打中文，切出去查个资料再切回来，
强行切成英文就是干扰。

`VimEnter` 那条也有讲究：

```lua
vim.api.nvim_create_autocmd('VimEnter', {
  group = group,
  pattern = '*',
  callback = function()
    -- VimEnter has no key source to attribute. In tmux, wait for a
    -- client-scoped event instead of changing the desktop IM on startup.
    if not vim.env.TMUX_PANE or vim.env.TMUX_PANE == '' then
      vim.schedule(switch_to_en_for_client)
    end
  end,
})
```

**在 tmux 里启动时不切。** 理由是启动这个动作没有"按键来源"可归因——下一节讲。

### `<Esc>` 映射：利用编码的有损性

```lua
-- Letters typed while the IM is composing are consumed by the IM and
-- never reach nvim, so they cannot be remapped. Pressing <Esc> cancels
-- the composition; the next <Esc> lands here and resets the IM.
map({ 'n', 'x' }, '<Esc>', function()
  switch_to_en_for_client()
  return '<Esc>'
end, { expr = true })
```

这条映射有一个漂亮的副作用。回忆 [第三篇](03-terminal-keys.html)：

> `Ctrl+[` 和 `Esc` 都编码成 `0x1b`，程序无法区分。

在别的场景下这是个缺陷，但在这里**它是免费的覆盖**：为 `<Esc>` 写的逻辑自动覆盖了
`Ctrl+[`，不需要额外一条映射。设计文档里明确记了这一点：

> Ctrl-[ 发送 `0x1b`，与 Esc 键不可区分，任何 Esc 路径天然覆盖它；真正不覆盖的
> 只有 Ctrl-C 和 Nvim 内部命令退出（`:stopinsert` 等），按一次 Esc 可自愈。

最后半句是关键：**没覆盖到的路径，用户按一次 Esc 就能自愈**。设计一个"尽力而为"的
机制时，重要的不是覆盖 100%，而是**未覆盖的情况有低成本的自愈路径**。

### 全角标点：让错误变成非错误

```lua
local fullwidth_keys = {
  ['：'] = ':',
  ['？'] = '?',
  ['／'] = '/',
  ['，'] = ',', -- leader
}
for fw, half in pairs(fullwidth_keys) do
  map({ 'n', 'x' }, fw, function()
    switch_to_en_for_client()
    return half
  end, { expr = true, remap = true })
end
```

[第十一篇](11-neovim.html) 讲过。补充一点**为什么只有标点能这么做**：

代码注释说得很清楚：

> Full-width punctuation commits instantly (no candidate window), so it
> reaches normal mode as a multibyte keypress.

全角标点在 rime 里是**直接上屏**的（不进候选窗口），所以它作为一个多字节按键真的
到达了 Neovim，可以被映射。而字母键会进入 rime 的组字状态，**根本不会到达
Neovim**，所以无法用同样的手法处理。

设计文档里还强调了一句：

> 全角标点映射里的 `switch_to_en()` 必须保留——否则矫正了标点却留中文态，后续按键
> 仍被 rime 吞。

**只纠正症状不纠正状态，问题下一秒就会重现。**

### 后端 fallback 链

```lua
fallback = fcitx_switcher 'fcitx5-remote'
if not fallback then
  fallback = fcitx_switcher 'fcitx-remote'
end
if not fallback then
  fallback = xdotool_switcher()
end
```

三级：

```text
busctl (首选)  ──失败──▶  fcitx5-remote / fcitx-remote  ──失败──▶  xdotool (需 opt-in)
```

`fcitx5-remote` 那一级要先查状态再决定：

```lua
for line in (result.stdout or ''):gmatch '[^\r\n]+' do
  if vim.trim(line) == '2' then -- 简体中文输入状态
    vim.system({ executable, '-c' }, { env = environment })
    return
  end
end
```

`xdotool` 那一级发的正是前面那个冷僻组合键：

```lua
-- xdotool sends synthetic X11 keyboard input, which can wake a DPMS-off
-- monitor. Keep it opt-in even on local graphical sessions.
local command = { executable, 'key', 'ctrl+alt+shift+F12' }
```

**它默认关闭，要 `NVIM_ALLOW_XDOTOOL_IM_SWITCH=1` 才启用。** 原因写在注释里：合成
X11 按键会唤醒已经 DPMS 关闭的显示器。

这个细节很能说明问题：**一个"自动化"如果会在你不看屏幕的时候把屏幕点亮，那它的
干扰远大于收益。** 而且它还有 [第六篇](06-who-gets-the-key.html) 案例三那个问题——
合成按键会重走责任链。所以它排在最后一级且默认关闭。

### 节流

```lua
local function throttle(fn)
  if not uv or not uv.now then
    return fn
  end

  local last = 0
  return function(...)
    local now = uv.now()
    if last ~= 0 and now - last < 200 then
      return
    end
    last = now
    return fn(...)
  end
end
```

200ms 内只执行一次。因为 `InsertLeave` 这类事件在快速操作时会密集触发，每次都
fork 一个 `busctl` 进程代价太高。

**幂等 + 节流是一对**：因为动作幂等，丢掉中间的调用不影响最终状态；因为节流，
不会产生进程风暴。

## 硬核部分：tmux 多 client 归因

前面都还算直接。真正的难题在这里。

### 问题

一个 tmux session 可以被多个 client 同时 attach：

```text
┌─ Debian 本地图形桌面 ─┐        ┌─ macOS 通过 ssh ─┐
│  WezTerm              │        │  Terminal.app     │
│  tmux attach          │        │  tmux attach      │
└───────────┬───────────┘        └────────┬──────────┘
            │                             │
            └──────────┬──────────────────┘
                       ▼
              同一个 tmux session
                       │
                  Neovim 进程
```

Neovim 只有一个，但按键可能来自任何一个 client。**如果按键来自 macOS 那台，
Neovim 就不该去改 Debian 本地的输入法。**

### 陷阱一：pane 环境是冻结的

最直觉的做法是看 `$SSH_CLIENT` 之类的环境变量。**这是错的**，而且我踩过：

> pane 环境在创建时冻结：本机 pane 残留 `SSH_CLIENT`/`SSH_TTY`，是 f529e2f 误判的
> 根因。

pane 继承的是**创建它时**的环境。如果这个 pane 是从 ssh 会话里创建的，之后你从本地
attach 同一个 session，pane 里的 `SSH_*` 还在——Neovim 会误判成远程，不切输入法。

反过来也一样：本地创建的 pane 没有 `SSH_*`，从 ssh attach 后按键，Neovim 会误判成
本地，去改远端机器的输入法。

**教训：长生命周期的进程不能用自己的环境变量判断"当前连接方式"。**

### 陷阱二：`client_pid` 会全局回退

第二个直觉是问 tmux："当前 client 是谁？"

```console
$ tmux display-message -p '#{client_pid}'
```

**也是错的**：

> `display-message -p '#{client_pid}'` 对 client 类 format 是**全局**回退到最近
> 活动 client：detached 会话里执行会返回**别的会话**的 client PID（不是空串）。
> 不能用它做 per-session 归因。

在一个 detached 的 session 里执行，它会返回**另一个 session 的 client**。这种
"返回了一个看起来合理但完全错误的值"是最难查的一类 bug。

### 正确做法

```lua
-- target resolver maps a pane id (%...) to the best session containing its
-- window (a linked window can belong to several sessions). Let
-- tmux compare activity timestamps instead of parsing a display format:
-- activity order puts the newest client first.
local function tmux_active_client_command(pane)
  return { 'tmux', 'list-clients', '-t', pane, '-O', 'activity', '-F', '#{client_pid}' }
end
```

三个要点：

1. **`list-clients -t <pane>`** 是**作用域受限**的：只列出能看到这个 pane 的
   client。detached 时正确返回空。
2. **`-O activity`** 按活动时间排序，最新的在前。文档明确警告过不要加 `-r`：

   > `-O activity` already puts the most recently active client first。
   > Do not add `-r`; it reverses the order and puts the least recently active
   > client first.

3. **`-F '#{client_pid}'`** 直接输出 PID，不用解析复杂格式。

拿到 PID 之后，读它的**实时**环境：

```lua
local path = string.format('/proc/%d/environ', pid)
```

> client 进程（`tmux a`）每次 attach 新建，`/proc/<pid>/environ` 永远新鲜。

**这是整个方案的关键洞察**：pane 的环境是冻结的，但 **client 进程的环境永远是新鲜
的**，因为 client 是每次 attach 时新建的进程。

拿到 client 环境后，还要**用它去执行 D-Bus 调用**：

```lua
local im_client_environment_names = {
  'DBUS_SESSION_BUS_ADDRESS',
  'DISPLAY',
  'WAYLAND_DISPLAY',
  'XDG_RUNTIME_DIR',
  'XAUTHORITY',
  'PATH',
}
```

```lua
local environment = {}
for _, name in ipairs(im_client_environment_names) do
  -- Empty values deliberately mask stale values inherited from the Nvim
  -- pane when a freshly attached client does not provide that variable.
  environment[name] = client_environment[name] or ''
end
```

注意那个 `or ''`：**如果 client 没有某个变量，显式设成空串来遮蔽 Neovim 自己继承
的旧值**。不这么做的话，ssh client 的调用会用上 pane 里残留的本地 `DISPLAY`，
又变成误改本地输入法。

### 一个不能用的判据

```text
本地 client 无连接类变量，但含 SSH_AUTH_SOCK / SSH_AGENT_PID（agent 转发），
这两个变量不能用于判别。
```

因为 ssh agent 转发会让**本地**进程也带上 `SSH_AUTH_SOCK`。判据只能用
`SSH_CLIENT` / `SSH_CONNECTION` / `SSH_TTY` / `MOSH_*`：

```lua
local remote_env_names = { 'SSH_CLIENT', 'SSH_CONNECTION', 'SSH_TTY', 'MOSH_IP', 'MOSH_CONNECTION' }
```

### 显式标记优先

启发式再准也有边界（比如嵌套 tmux），所以留了显式逃生舱：

```lua
local function classify_remote_environment(environment)
  local marker = environment.TMUX_IM_CLIENT
  if marker == 'local' then
    return false
  end
  if marker == 'ssh' or marker == 'remote' then
    return true
  end
  -- 否则走 SSH_*/MOSH_* 启发式
```

用法：

```sh
# Local desktop terminal
TMUX_IM_CLIENT=local tmux attach -t work

# SSH terminal
TMUX_IM_CLIENT=ssh tmux attach -E -t work
```

文档特别提醒：

> The marker must be set on the tmux client process. Do not use
> `tmux set-environment` for this purpose because that environment is shared by
> the whole session.

**必须设在 client 进程上，不能用 `tmux set-environment`**——后者是整个 session
共享的，正好是我们要避开的那个东西。

### 竞态：为什么可以接受

整个查询链（`list-clients` → 读 `/proc`）是异步的，理论上存在竞态：查询在飞的时候
另一个 client 按了键。分析结论是：

> `client_activity` 在 server 收到按键时更新，先于按键分发到 pane；Nvim 的异步查询
> 必然晚于此刻，所以**触发事件的 client 按定义就是 activity 最新者**。剩余竞态
> （查询在飞期间另一 client 按键）毫秒级，且最终动作幂等，无害。

**又回到幂等。** 因为最坏情况只是"多设一次已经是那个值的状态"，所以毫秒级竞态
不需要用锁去消除。文档也诚实标注了这一点：

> It is a best-effort attribution mechanism, not an atomic association between
> one tmux key event and one Neovim callback.

### 失败时默认切

```text
d. 任何一步失败（进程已退出、/proc 不可读、tmux 报错）→ 默认 local（切）
```

理由：

> 默认 local 的理由：busctl/fcitx5-remote 路径误判代价只是幂等地设一次 ASCII
> mode；唯一有真实副作用的 xdotool 已被 `NVIM_ALLOW_XDOTOOL_IM_SWITCH` 独立
> opt-in 挡住。

**默认值的选择依据是"误判的代价"，不是"猜对的概率"。** 这是设计容错策略时很好的
思路。

### 可覆盖

```text
NVIM_DISABLE_IM_SWITCH=1        全部关闭
NVIM_ALLOW_REMOTE_IM_SWITCH=1   允许远程 client 触发
NVIM_ALLOW_UNKNOWN_IM_SWITCH=1  允许无法归因时触发
NVIM_ALLOW_XDOTOOL_IM_SWITCH=1  允许合成按键 fallback
```

**任何"自动"机制都应该有一个明确的关闭开关。** 自动化在 95% 的情况下帮忙，在 5%
的情况下碍事——如果那 5% 无法关闭，用户对整个机制的信任就没了。

## 三个被否决的方案

记录否决理由跟记录最终方案同样重要。

### 否决一：让 rime 处理 `Alt_L`

```yaml
patch:
  ascii_composer/switch_key/Alt_L: clear
```

想法是：让 rime 自己在检测到"单独按下左 Alt"时切英文。

> 该配置可以进入 Rime build 产物，但在 WezTerm + fcitx5-rime 的路径里，
> Rime 没有可靠收到"单独按下左 Alt"这个事件。

根因在 [第三篇](03-terminal-keys.html)：**修饰键本身不产生任何字节，"单独按下修饰
键"这个事件在很多路径上根本传不到**。

### 否决二：tmux root key binding

```tmux
set -g prefix None
bind-key -n M-b run-shell ... \; switch-client -T prefix
```

否决理由有两条，第二条是踩过的坑：

> - 它不再是 tmux 原生 prefix，只是临时切换 key table，时序和容错都更差。
> - 如果在这里用 `xdotool --clearmodifiers` 模拟按键，会释放/恢复 Alt，干扰
>   Awesome WM 看到的修饰键状态，导致 `Alt+b` 后的 `j` 有时变成 WM 的
>   `Alt+j` 快捷键。

第一条是通用原则：**用原生机制 + 上游预处理，优于重新实现一个近似的状态机。**

### 否决三：tmux 全局拦截 Esc

```tmux
bind-key -n Escape { if-shell -F '#{==:#{@nvim_im_switch},1}' \
  { run-shell -b 'tmux-im-escape #{client_pid}' } ; send-keys Escape }
```

理论上归因最准（tmux 知道是哪个 client 按的），但：

> - 全局拦截所有 pane 的 Esc，每次按键 fork helper（2~4 进程）
> - `extended-keys csi-u` 下 Esc 经 CSI-u 解码路径，交互未实测
> - 要拆掉 Nvim 侧全部非 Esc 事件的切换，Ctrl-C 退出后中文态残留
> - helper 脚本、tmux.conf、Nvim 标记三处维护，跨两个仓库；helper 缺失时静默退化

最后一条是决定性的：**跨三个地方、两个仓库的维护成本，换来的是毫秒级竞态的消除——
而那个竞态因为幂等本来就无害。**

## 环境变量污染

最后一个必须知道的坑，[第六篇](06-who-gets-the-key.html) 提过，这里补完整。

**症状**：

> - zsh 新行不再自动把 Rime 切到英文输入模式
> - 打开 Neovim 时提示 `Failed to switch input method`
> - 从另一个终端执行一次 `tmux a` 后，新 pane 又突然恢复正常

**根因**：`update-environment` 在 attach 时会用 client 环境刷新 session 环境；
**如果 client 没有某个变量，tmux 会把它标记为 removed**。从 ssh attach 一次，本地
图形会话的 `DISPLAY` 就被清掉了。

**诊断**：

```console
$ tmux show-environment DISPLAY
-DISPLAY          ← 减号前缀 = 已标记移除
```

**规避**：

```console
# ssh attach 用 -E 跳过 update-environment
$ tmux attach -E

# 或者本地和 ssh 用不同 socket（更彻底）
$ tmux new -A -s work            # 本地
$ tmux -L ssh new -A -s ssh      # ssh
```

**注意 Neovim 那套 per-client 方案天然免疫这个问题**——它不读 session 环境，而是
直接读 client 进程的 `/proc/<pid>/environ`。这是那个设计的一个额外收益：

> This design does not add a root Escape binding or depend on tmux's session-wide
> environment. The existing tmux client environment is read directly, so no
> DBus variables need to be copied into `update-environment` for this feature.

zsh 那套仍然受影响，因为它用的是自己进程的环境。

## 三处对比

| | zsh ZLE | WezTerm | Neovim |
| --- | --- | --- | --- |
| **触发** | `line-init` 且命令行为空 | 按下 `Alt+b` | InsertLeave / CmdlineLeave / TermLeave / FocusGained / Esc / 全角标点 |
| **归因** | 靠自己进程的环境变量 | 构造性正确（只有本地 WezTerm 会执行） | per-client 查 `/proc/<pid>/environ` |
| **后端** | busctl | busctl | busctl → fcitx5-remote → xdotool |
| **异步** | 否（同步，但很快） | 否 | 是（`vim.system` + 200ms 节流） |
| **ssh 安全** | 靠 `DISPLAY`/`DBUS` 守卫 | 天然（不经过） | 显式 per-client 判定 |
| **复杂度** | 20 行 | 10 行 | ~250 行 |

**复杂度的差异全部来自归因难度。** zsh 和 WezTerm 都能从自己的位置直接确定"这个
事件是本地产生的"；Neovim 在 tmux 里跑，跟按键来源之间隔了一个 tmux server，所以
必须显式地把来源查出来。

**这也是一条通用经验：自动化的复杂度，往往不在动作本身，而在"确定该不该做这个
动作"。**

## 设计原则总结

从这三处实现里能提炼出的通用原则：

1. **状态自动恢复优于状态可见。** 让问题不需要被问出来，比让答案更容易看到更好。
2. **动作做成幂等的设置，不要做成 toggle。** 这能省掉整类同步问题，让"尽力而为"的
   架构成立。
3. **先查询再设置。** 无条件设置会触发状态提示，让功能自己变成干扰源。
4. **在最知道上下文的层实现。** ZLE 知道"shell 在等命令"，WezTerm 知道"这是本地
   按键"，这些信息在别的层拿不到。
5. **上游预处理优于下游改造。** WezTerm 截获转发，tmux 一行不改。
6. **未覆盖的情况要有低成本自愈路径。** 按一次 Esc 就好，比追求 100% 覆盖更实际。
7. **默认值按"误判代价"选，不按"猜对概率"选。**
8. **让错误的输入产生正确的结果**（全角标点映射），优于检测错误再提示。
9. **任何自动机制都要有关闭开关**，否则 5% 的碍事会毁掉 95% 的信任。
10. **长生命周期进程不能用自己的环境变量判断当前连接方式**；要查实时的、每次新建的
    那个进程。

下一篇是系列收束，把 fcitx5 的极简配置和整套键位哲学讲完：
[fcitx5 极简配置与键位设计哲学](13-philosophy.html)。
