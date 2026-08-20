---
title: "Linux 按键体系 10：命令行层 zsh ZLE"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第十篇。
> 责任链的第 ⑦ 层，zsh 的 Zsh Line Editor（ZLE）。这是字节流的终点：从这里开始，
> "按键"重新变回可以被绑定的东西。

## ZLE 是一棵前缀树

ZLE 从终端读到的是字节序列（[第三篇](03-terminal-keys.html)）。它维护一棵**按键
序列前缀树**，`bindkey` 就是往树里插节点：

```console
$ bindkey | head -8
"^@" set-mark-command
"^A" beginning-of-line
"^B" backward-char
"^D" delete-char-or-list
"^E" end-of-line
"^[[A" up-line-or-beginning-search
"^[OA" up-line-or-beginning-search
```

左边是字节序列（`^A` = `0x01`，`^[` = `ESC`），右边是 **widget** 名。

树的结构带来一个直接后果：**当输入的前缀能匹配多个更长的序列时，ZLE 必须等**。
比如收到 `ESC` 后，它不知道你是按了 Esc 键，还是 `Alt+某键` 的第一个字节，只能等
`KEYTIMEOUT`（默认 40，单位是 1/100 秒，即 400ms）。

这跟 [第九篇](09-tmux.html) 的 `escape-time` 是同一个问题在不同层的复现。emacs 模式
下影响不大（Esc 本身没绑什么），但如果你用 `bindkey -v`（vi 模式），每次按 Esc 退出
插入模式都要等 400ms——那时就必须调小：

```zsh
KEYTIMEOUT=1    # 10ms
```

## 为什么命令行用 emacs 模式

我的编辑器是 Neovim，但 zsh 用的是 emacs 键位。这看起来矛盾，实际上是一个明确的
取舍。

oh-my-zsh 的 `lib/key-bindings.zsh` 里：

```zsh
bindkey -e
```

**理由是模态本身有成本。** 回到系列主线的第二条标准——减少状态确认：

- Vim 的模态在编辑**几百行代码**时收益巨大：你有大量的移动、重复、批量操作，
  normal 模式的动词-名词语法能极大压缩击键。
- 命令行**通常只有一行**。移动距离短，批量操作几乎没有。而每输入一条命令都要经历
  "插入 → Esc → 移动 → i → 继续输入"的模式切换，同时还要维护"我现在在哪个模式"
  这个状态。

在一行文本上，**模态的管理成本超过了它的编辑收益**。

而且 emacs 键位在这里恰好很划算，因为 [第四篇](04-xmodmap.html) 把 Ctrl 搬到了
拇指位置：

| 键 | 动作 | 使用频率 |
| --- | --- | --- |
| `Ctrl+a` / `Ctrl+e` | 行首 / 行尾 | 极高 |
| `Ctrl+w` | 向前删一个词 | 极高 |
| `Ctrl+u` / `Ctrl+k` | 删到行首 / 行尾 | 高 |
| `Ctrl+r` | 历史搜索 | 极高 |
| `Ctrl+y` | 粘回删掉的内容 | 中 |

这些全部是"按住拇指 + 一个字母"，没有模式切换，没有状态。

**这是一个值得记住的判断标准：模态的收益随编辑规模增长，成本随切换频率增长。**
一行命令的场景下，成本占优。

还有一个很实际的理由：Vim 模式下 zsh 的默认视觉反馈几乎没有——你无法一眼看出自己
在 normal 还是 insert 模式。要补上这个（改光标形状或 RPROMPT 指示器）需要额外配置：

```zsh
# 如果确实要用 vi 模式，至少补上状态指示
function zle-keymap-select {
  case $KEYMAP in
    vicmd)      print -n '\e[2 q' ;;   # 方块光标 = normal
    viins|main) print -n '\e[6 q' ;;   # 竖线光标 = insert
  esac
}
zle -N zle-keymap-select
```

**没有这段配置的 vi 模式是"逼迫用户反复确认状态"的典型反面教材**：你必须先试着按
一个键，看它是插入还是移动，才知道自己在哪个模式。

## terminfo：正确绑定特殊键的唯一方式

[第三篇](03-terminal-keys.html) 讲过方向键有 normal / application 两套编码。
oh-my-zsh 的处理方式是教科书级的：

```zsh
# Make sure that the terminal is in application mode when zle is active, since
# only then values from $terminfo are valid
if (( ${+terminfo[smkx]} )) && (( ${+terminfo[rmkx]} )); then
  function zle-line-init() {
    echoti smkx
  }
  function zle-line-finish() {
    echoti rmkx
  }
  zle -N zle-line-init
  zle -N zle-line-finish
fi

bindkey -e

if [[ -n "${terminfo[kcuu1]}" ]]; then
  bindkey -M emacs "${terminfo[kcuu1]}" up-line-or-beginning-search
  bindkey -M viins "${terminfo[kcuu1]}" up-line-or-beginning-search
  bindkey -M vicmd "${terminfo[kcuu1]}" up-line-or-beginning-search
fi
```

三件事，缺一不可：

1. **`zle-line-init` 里 `echoti smkx`** —— 每次开始编辑命令行时，主动把终端切到
   application 模式。
2. **绑定时用 `$terminfo[kcuu1]`** —— 从 terminfo 数据库取序列，而不是硬编码
   `'^[[A'`。
3. **`zle-line-finish` 里 `echoti rmkx`** —— 编辑结束后切回来，免得影响接下来运行
   的程序。

**为什么必须这么麻烦？** 因为 terminfo 里记的是 application 模式的序列
（`kcuu1=\EOA`）。如果不主动切模式，终端发的是 `ESC [ A`，而你绑的是 `ESC O A`，
两边对不上。

硬编码 `bindkey '^[[A' ...` 在你自己的机器上"能用"，但换个 `TERM`、ssh 到别的机器，
就会莫名其妙失效。**这是配置可移植性的一个典型分水岭。**

## hook：为什么要用 `add-zle-hook-widget`

`zle-line-init` 是个特殊 widget，ZLE 在每次开始编辑新命令行时调用它。上面 oh-my-zsh
已经用 `zle -N zle-line-init` 占了这个位置。

我要在同一时机做另一件事（把 rime 切回英文）。如果直接写：

```zsh
# 错误示范
function zle-line-init() { __rime_ensure_ascii_mode }
zle -N zle-line-init
```

**oh-my-zsh 那个 `echoti smkx` 就被覆盖掉了**，方向键立刻全部失灵。而且这个故障
非常难查——你改的是输入法，坏的是方向键，两件事看起来毫无关系。

正确做法是用 zsh 自带的 hook 机制：

```zsh
# home/.zshrc
if [[ -o interactive ]]; then
  autoload -Uz add-zle-hook-widget
  add-zle-hook-widget line-init __rime_ascii_mode_on_empty_zle_line
fi
```

`add-zle-hook-widget` 维护一个 widget **列表**，逐个调用。多个来源可以挂同一个
hook，互不干扰。

**这是配置可长期维护的一条硬规则：涉及共享扩展点时，永远用框架提供的注册机制，
不要直接赋值覆盖。** 同样的规则适用于：

| 场景 | 不要 | 要 |
| --- | --- | --- |
| zsh 命令行 hook | `zle -N zle-line-init` | `add-zle-hook-widget line-init` |
| zsh precmd | `precmd() {...}` | `add-zsh-hook precmd my_func` |
| Neovim autocmd | 裸 `autocmd` | `nvim_create_augroup` + `clear = true` |
| tmux hook | `set-hook` | `set-hook -a`（append） |

## 那个 rime hook

既然讲到了，把完整实现放这里（原理见 [第十二篇](12-rime-auto-switch.html)）：

```zsh
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
```

注意 `__rime_ascii_mode_on_empty_zle_line` 里那个条件：**只在 `BUFFER`、`LBUFFER`、
`RBUFFER` 全空时才切换**。

也就是说：新 prompt 出现、命令行是空的 → 切英文；已经在编辑（比如光标停在
`echo ` 后面准备输入中文）→ 不动。

**这个边界是刻意画的。** 新命令行几乎总是以英文命令开头，自动切英文对用户永远
是对的；而一旦开始编辑，用户可能正需要中文，强行切换就成了干扰。

再看三个前置检查——没有 D-Bus、没有图形会话、没有 `busctl` 就直接返回。这让同一份
`.zshrc` 在 ssh 会话、tty、容器里都能安全加载，不会报错也不会误改远端状态。
**"可移植"在实践中往往就是这样一串 guard clause。**

## 为什么 ZLE 是放这个逻辑的正确层

自动切输入法可以放在很多地方，我最终选了 ZLE，理由记在
`config/wezterm/docs/rime-tmux-prefix.md` 里：

> 1. ZLE 只在 shell 正在编辑命令行时运行，因此不会影响 `codex`、`nvim`、`fzf`
>    等子进程自己的输入界面。

这是**作用域精确性**的胜利。对比一下备选方案：

| 放在哪 | 问题 |
| --- | --- |
| WezTerm（终端层） | 分不清当前 pane 里跑的是 shell 还是 Neovim，会误切 |
| tmux | 同上，而且要拦截 root key，代价大 |
| **zsh ZLE** | **只在 shell 编辑命令行时触发，精确** |

一条通用原则：**把逻辑放在"最知道当前上下文"的那一层**。ZLE 知道"现在是 shell 在
等你输命令"，这个信息在上面任何一层都拿不到。

## 少量的显式绑定

我自己加的 `bindkey` 只有一条：

```zsh
bindkey '^N' delete-word
```

`Ctrl+n` 默认是"下一行历史"，我改成了删词。

**为什么只有一条？** 因为 emacs 默认键位加上 oh-my-zsh 的 terminfo 绑定已经覆盖了
绝大部分需求。**改得越少，跟别人的机器、跟未来的自己的差异就越小。**

这条原则在系列主线的第三条标准（可长期维护）下尤其重要：每一条自定义绑定都是一份
需要记忆和维护的负债。加之前先确认默认键位真的不够用。

## 与其他工具的键位分配

zsh 层最有意思的部分其实是**跟外部工具协商键位**。

### fzf

```zsh
if [[ -t 0 ]] && (( $+commands[fzf] )); then
  source <(fzf --zsh)
  export FZF_DEFAULT_OPTS="--bind='tab:down,shift-tab:up' --cycle"
fi
```

`fzf --zsh` 注册三个绑定：

| 键 | 作用 |
| --- | --- |
| `Ctrl+r` | 模糊搜索历史 |
| `Ctrl+t` | 模糊选文件插入命令行 |
| `Alt+c` | 模糊选目录并 cd |

`FZF_DEFAULT_OPTS` 里把 `Tab` 改成"下移一项"。默认 fzf 用 `Tab` 做多选标记，但我
更常用单选，`Tab`/`Shift+Tab` 上下移动跟补全菜单的直觉一致。`--cycle` 让列表首尾
相连，到底了继续按会回到开头——**消除"到头了"这个需要感知的状态**。

### atuin

```zsh
(( $+commands[atuin] )) && eval "$(atuin init zsh --disable-up-arrow)"
```

`--disable-up-arrow` 是一个刻意的键位分配决定：

```text
↑        → 保持 zsh 原生行为（按前缀搜索当前会话历史）
Ctrl+r   → 交给 atuin（跨会话、跨机器的全量历史 + 上下文过滤）
```

两种历史检索需求是不同的：**"我刚才那条命令"用 `↑` 最快**（不需要打开界面，不需要
输入查询）；**"我上个月在另一台机器上那条命令"用 `Ctrl+r`**。

如果让 atuin 同时接管 `↑`，每次想重复上一条命令都要弹出一个全屏界面——**用重型工具
解决轻量问题，是键位设计里很常见的一种退化。**

### fzf-tab

```zsh
zstyle ':fzf-tab:*' fzf-bindings 'space:accept'
zstyle ':fzf-tab:*(cat|ls)*' accept-line enter
zstyle ':fzf-tab:*' fzf-command ftb-tmux-popup
zstyle ':fzf-tab:complete:(cd|go):*' disabled-on any
```

fzf-tab 把 zsh 的补全菜单换成 fzf 界面。四条配置各有考究：

- **`space:accept`** —— 空格键接受当前选项。补全场景下你几乎总是要接着输入下一个
  参数，而参数之间本来就要打空格。**把"确认"和"下一步"合并成一个动作。**
- **`ftb-tmux-popup`** —— 补全菜单渲染在 **tmux 的浮动 popup** 里。这是一个漂亮的
  跨层协作：第 ⑦ 层的补全借用第 ⑥ 层的窗口能力，好处是不占用当前 pane 的滚动缓冲，
  关掉后布局完全不变（跟 [第九篇](09-tmux.html) 的 `prefix t` 同一个思路）。
- **`disabled-on any` for `cd`/`go`** —— 这两个命令的补全**不走** fzf。因为
  `cd` 的补全通常只有几个候选，直接用 zsh 原生菜单更快，弹一个 popup 反而是打断。

最后一条尤其值得注意：**好的工具集成不是"到处都用它"，而是知道什么时候不用它。**

### 粘贴性能

```zsh
# Keep large bracketed pastes responsive. Oh My Zsh's bracketed-paste-magic
# replays pasted text through ZLE widgets, which is much slower than zsh's
# native paste widget for large payloads.
if [[ -o interactive ]]; then
  zle -A .bracketed-paste bracketed-paste 2>/dev/null
fi
typeset -g ZSH_HIGHLIGHT_MAXLENGTH=3000
```

`zle -A` 是 widget 别名：把内置的 `.bracketed-paste`（前面那个点表示内置版本）
重新绑到 `bracketed-paste` 这个名字上，覆盖掉 oh-my-zsh 的
`bracketed-paste-magic`。

后者会把粘贴的文本**逐字符重放**过 ZLE widget 链（为了触发语法高亮、历史展开等），
在粘贴几百行时会卡住好几秒。

`ZSH_HIGHLIGHT_MAXLENGTH=3000` 是同一个问题的另一半：超过 3000 字符就不做语法高亮。

**这两条不是键位配置，但它们直接决定"按下 `Ctrl+Shift+v` 之后会不会卡住"。**
按系列的标准，一次几秒的卡顿比任何键位不便都更严重地打断心流——因为你不知道它是卡住
了还是死了，会开始怀疑、会想按 `Ctrl+c`，注意力完全离开原来的任务。

## 调试

```console
# 完整键位表
$ bindkey

# 某个序列绑到了什么
$ bindkey '^N'

# 查某个 widget 绑在哪些键上
$ bindkey | rg 'delete-word'

# 看某个键实际发出的字节：按 Ctrl+v 再按目标键

# 列出所有可用 widget
$ zle -la

# 当前 keymap（emacs / viins / vicmd）
$ bindkey -lL
```

## 小结

1. **ZLE 是按键序列前缀树**，前缀有歧义时要等 `KEYTIMEOUT`；vi 模式下必须调小。
2. **命令行用 emacs 模式**：模态的收益随编辑规模增长、成本随切换频率增长，一行
   命令的场景成本占优。真要用 vi 模式，必须补上光标形状指示。
3. **特殊键一律用 `$terminfo[...]` 绑定**，并在 `zle-line-init` 里 `echoti smkx`，
   否则换 `TERM` 就失效。
4. **共享扩展点用 `add-zle-hook-widget` 而不是直接 `zle -N` 覆盖**，否则会静默
   破坏别人的功能。
5. **把逻辑放在最知道上下文的层**：ZLE 知道"shell 正在等你输命令"，这个信息上层
   拿不到。
6. **跟外部工具协商键位时区分轻重**：`↑` 留给轻量的"上一条"，`Ctrl+r` 给重型的
   全量搜索；`cd` 的补全不套 fzf。

下一篇是最后一个应用层：
[编辑层：Neovim](11-neovim.html)。
