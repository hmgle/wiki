---
title: "Linux 按键体系 06：谁先拿到这个键"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第六篇，回答系列的第二个核心
> 问题：**ssh / awesome / wezterm / tmux / zsh / nvim 组合环境下，一个快捷键被谁
> 捕获、按什么顺序处理？**
>
> 这是整个系列的枢纽。前五篇讲"键是什么"，从本篇开始讲"键去了哪里"。

## 责任链

先把完整的捕获顺序摆出来。**每一层都可以吃掉按键，吃掉之后下层永远收不到。**

```text
① keyd / xmodmap        改写键的身份（不"吃"，只翻译）
        │
② AwesomeWM             XGrabKey 抢占的全局键 ── 吃掉 ──▶ 执行 WM 动作
        │ 未被 grab
③ fcitx5 + rime         中文态下的字母键 ────── 吃掉 ──▶ 变成候选词
        │ 透传
④ WezTerm               自己的 keys 表 ──────── 吃掉 ──▶ 执行终端动作
        │ 未匹配 → 编码成字节写入 PTY
⑤ termios 行规程        C-c / C-z / C-d ────── 转换 ──▶ 信号
        │ raw 模式下全部透传
⑥ tmux                  prefix + 绑定 ───────── 吃掉 ──▶ 执行 tmux 动作
        │ 未匹配 → 重新编码写入内层 PTY
⑦ zsh ZLE / Neovim      查自己的键位表
```

有几个反直觉的地方值得强调：

- **输入法在窗口管理器之后、终端之前。** 所以 WM 的全局键不受输入法状态影响，但
  终端及以下的所有层都受。这是 [第十二篇](12-rime-auto-switch.html) 整篇的由来。
- **termios 在终端和 tmux 之间。** 但 tmux、Neovim 这类全屏程序启动时会切到 raw
  模式关掉它，所以实践中它只对普通命令行程序生效。
- **tmux 处于双重身份**：对 WezTerm 它是"终端里的程序"，对 Neovim 它是"终端"。
  按键要被解码-再编码两次。

## 我的实际分配

光有顺序还不够，关键是**每一层占用了哪些键**。这是我当前的分配：

| 层 | 修饰键 / 前缀 | 物理位置 | 占用范围 |
| --- | --- | --- | --- |
| ② AwesomeWM | `Mod1` | 物理 Windows 键 | `Mod1` + 字母/数字/方向 |
| ② AwesomeWM | `Control` + 数字 | 物理 Alt 键 + 1–9 | 切换 tag |
| ④ WezTerm | `Ctrl+Shift+Space`（leader） | — | leader 后的键 |
| ④ WezTerm | `Ctrl+Shift+c/v/t` | — | 复制/粘贴/新 tab |
| ④ WezTerm | `Alt+1`–`Alt+6` | 物理 Win + 数字 | 切换 tab |
| ⑥ tmux | `Alt+b`（prefix） | 物理 Win + b | prefix 后的键 |
| ⑥ tmux | `Ctrl+g`（prefix2） | 物理 Alt + g | 同上，ssh 备用 |
| ⑦ Neovim | `,`（leader） | — | leader 后的键 |
| ⑦ zsh/Neovim | `Ctrl` + 字母 | 物理 Alt 键 | readline / 编辑操作 |

回忆 [第四篇](04-xmodmap.html) 那个交换：**物理 Alt 键产生 Ctrl 修饰键，物理
Windows 键产生 mod1（也就是各处显示的 "Alt"）**。所以：

```text
        ┌──────┬──────┬───────┬───────┬─────────────┐
键帽    │ Ctrl │  Fn  │ Super │  Alt  │    Space    │
├───────┼──────┼──────┼───────┼───────┼─────────────┤
实际    │ Ctrl │  Fn  │  Alt  │ Ctrl  │    Space    │
├───────┼──────┼──────┼───────┼───────┼─────────────┤
归谁用  │  ⑦   │      │  ②④⑥  │  ⑦②   │             │
        └──────┴──────┴───────┴───────┴─────────────┘
                  窗口/终端层    终端内程序层
```

**这个物理分离是整套设计的地基**：拇指底下那个键给终端内的程序用（Ctrl），
Windows 键给窗口管理器和终端本身用（mod1）。两类操作在手上就是两个不同的动作，
不需要在脑子里做"这个键现在归谁"的判断。

### 一个真实的取舍

AwesomeWM 用 `Control` + 数字切 tag，也就是物理 Alt + 1–9。代价是：**这些组合被
全局 grab 了，终端里永远收不到**。

而 `Ctrl+2`、`Ctrl+6`、`Ctrl+7`、`Ctrl+8` 在 ASCII 里是有意义的（`NUL`、`RS`、
`US`、`DEL`）。所以严格说，这个选择牺牲了四个几乎没人用的控制字符。

我认为这笔交易划算，但它说明一件事：**全局 grab 从来不是免费的**。第七篇会展开
这个话题。

## 冲突诊断：从下往上二分

"我按了某个键，什么也没发生"——这是最常见的报障。正确的排查方式不是改配置试，
而是**从最底层开始，逐层确认信号还在不在**。

### 标准流程

```text
第 1 步：X 层收到了吗？
  $ xev -event keyboard
  按下目标键，看有没有 KeyPress，keycode 和 keysym 对不对
  ✗ 没有 → 问题在 ① 层（keyd/xmodmap 映射错了，或键盘硬件问题）
  ✓ 有   → 继续

第 2 步：被窗口管理器抢了吗？
  按 Mod1+s 打开 awesome 的快捷键帮助，搜索这个组合
  或者：在 rc.lua 里 rg 一下
  ✗ 被绑了 → 问题在 ② 层
  ✓ 没绑   → 继续

第 3 步：被输入法吃了吗？
  切到英文状态再按一次
  ✗ 英文下正常 → 问题在 ③ 层（见第十二篇）
  ✓ 两种状态都不行 → 继续

第 4 步：终端发出字节了吗？
  $ cat -v
  按下目标键
  ✗ 没有输出 → 问题在 ④ 层（WezTerm 自己绑了，或没有对应编码）
  ✓ 有输出   → 记下这串字节，继续

第 5 步：tmux 吃了吗？
  临时退出 tmux（或 tmux detach），在裸终端里重试
  ✗ 裸终端正常 → 问题在 ⑥ 层
  ✓ 都不行     → 问题在 ⑦ 层

第 6 步：应用层
  zsh:    $ bindkey | rg '<字节序列>'
  nvim:   :verbose map <键>      ← verbose 会告诉你是哪个文件绑的
```

### 三个加速技巧

**一、`cat -v` 是分水岭。** 它把"图形层的问题"和"终端层的问题"一刀切开。第 4 步
输出正常，说明 ①②③④ 都没问题，可以直接跳到 tmux 和应用层。

**二、`Ctrl+v` 前缀在几乎所有地方都能用。** 在 zsh、tmux copy-mode、Neovim 插入
模式里，`Ctrl+v` 后跟一个键会显示它的原始字节。不用切窗口就能看。

**三、`:verbose map` 是 Neovim 里最有用的命令。** 它直接告诉你这个映射定义在哪个
文件的哪一行——比在配置里全文搜索快得多。

## 四个真实案例

### 案例一：中文状态下 tmux prefix 失效

**现象**：按 `Alt+b` 进入 tmux prefix，再按 `j`，没有切换 pane，而是弹出了拼音候选。

**定位**：第 3 步就能确认——切到英文状态一切正常。问题在 ③ 层。

**原因**：`Alt+b` 本身带修饰键，rime 不感兴趣，透传给了 tmux，tmux 进入 prefix
等待状态。但接下来的 `j` 是个裸字母键，中文态下 rime 直接把它吃掉当拼音了，tmux
永远等不到它的命令键。

**修法的选择**：可以在 ③ 层解决（让 rime 别吃），也可以在 ④ 层解决（终端截获
`Alt+b` 时先把 rime 切成英文再转发）。我选了后者，完整推理见
[第十二篇](12-rime-auto-switch.html)。

**这个案例的价值在于**：它是"跨层状态耦合"的典型——tmux 的 prefix 状态机和 rime 的
中英文状态机互不知情，但物理上共享同一串按键。

### 案例二：`Alt+j` 在 Neovim 里绑不上

**现象**：在 Neovim 里给 `<A-j>` 绑了个功能，按下去却切换了窗口。

**定位**：第 2 步。`rg 'modkey.*"j"' rc.lua` 就能看到：

```lua
awful.key({ modkey }, "j", function()
    awful.client.focus.byidx(1)
end, { description = "focus next by index", group = "client" }),
```

**原因**：AwesomeWM 用 `XGrabKey` 抢占了 `Mod1+j`，它根本到不了 Neovim。

**修法**：要么换个键，要么在 awesome 里放弃这个绑定。**不能两边都要。**

这类冲突的预防办法是**约定修饰键的归属**——见本文最后一节。

### 案例三：`xdotool` 引发的幽灵按键

这是我踩过的一个隐蔽的坑，记在 `config/wezterm/docs/rime-tmux-prefix.md` 里：

> 如果在这里用 `xdotool --clearmodifiers` 模拟按键，会释放/恢复 Alt，干扰
> Awesome WM 看到的修饰键状态，导致 `Alt+b` 后的 `j` 有时变成 WM 的 `Alt+j` 快捷键。

**原因**：`xdotool --clearmodifiers` 的工作方式是——先发送"抬起所有修饰键"的合成
事件，执行完目标动作，再发送"按下"恢复。但这些合成事件走的是 XTEST 扩展，**在
② 层也是可见的**。于是在那个短暂窗口里，AwesomeWM 认为 Alt 被重新按下了，而你手上
的 `j` 正好落在这个窗口里，就被解释成了 `Mod1+j`。

**教训**：**合成按键（`xdotool`、`xte`、XTEST）会从 ② 层重新走一遍完整的责任链**，
它不是"直接把字节塞给目标程序"。任何用合成按键做的自动化，都要考虑它会不会触发
上层的全局快捷键。这也是我最终在 WezTerm 里用 `SendKey` 而不是 `xdotool` 的原因：
`SendKey` 只影响 WezTerm 自己的 pane，不经过 X。

### 案例四：`Ctrl+i` 绑不开

**定位**：第 4 步。`cat -v` 显示 `Ctrl+i` 和 `Tab` 输出完全一样。

**原因**：④ 层的编码是有损的，见 [第三篇](03-terminal-keys.html)。信息在编码时就
丢了，⑥⑦ 层无论如何恢复不出来。

**这个案例的意义**：**不是所有冲突都能通过"换个层解决"**。有些是信息论意义上的
丢失，只能换编码（CSI u）或者换键。

## ssh：把责任链拉长一倍

ssh 是最容易搞混的场景，因为它让**两套完整的责任链串联**起来：

```text
┌─── 本地机器 ────────────────────────────────┐
│ ① keyd/xmodmap                              │
│ ② AwesomeWM        ← 全局键在这里就被吃掉    │
│ ③ fcitx5 + rime    ← 输入法在本地！          │
│ ④ WezTerm          ← 本地终端的键位          │
│ ⑥ 本地 tmux（如果有）                        │
│         │                                    │
│         └─ ssh 客户端 ── 字节流 ──┐          │
└───────────────────────────────────┼──────────┘
                                    │
┌─── 远端机器 ───────────────────────┼──────────┐
│      sshd ── PTY ──────────────────┘          │
│ ⑤ 远端 termios                                │
│ ⑥ 远端 tmux                                   │
│ ⑦ 远端 zsh / Neovim                           │
└───────────────────────────────────────────────┘
```

### 五条必须记住的规则

**一、①②③④ 全部在本地发生。** 你"人在远端"只是错觉——按 `Mod+Return` 打开的是
**本地**终端；输入法状态是**本地**的；WezTerm 的 leader 键是**本地**处理的。远端
程序完全不知道这些层存在。

**二、这意味着输入法自动切换在 ssh 场景下必须由本地负责。** 远端的 Neovim 通过
D-Bus 去改输入法状态，改的是**远端机器**的输入法——如果远端根本没有图形会话，
这个调用就是无效的；更糟的是如果远端也有人在用图形桌面，你会改掉别人的状态。
[第十二篇](12-rime-auto-switch.html) 里有一整套机制处理这个归因问题。

**三、tmux 套 tmux 需要区分 prefix。** 本地 tmux 和远端 tmux 如果 prefix 相同，
本地那个会先吃掉。两个解法：给远端用不同 prefix，或者按两次 prefix（tmux 的
`send-prefix` 会把 prefix 原样转发给内层）。我的配置里 `bind M-b send-prefix`
就是干这个的：

```tmux
set -g prefix M-b
set -g prefix2 C-g   # for macOS
unbind C-b
bind M-b send-prefix
```

**四、`TERM` 和 terminfo 必须在远端也存在。** 本地 `TERM=tmux-256color`，如果远端
没有这个 terminfo 条目，方向键、功能键的转义序列就对不上，表现为"按方向键出乱码"。
排查：

```console
$ echo $TERM
$ infocmp "$TERM" >/dev/null && echo ok || echo "远端缺少这个 terminfo"
```

修法是在远端安装 `ncurses-term`，或者用 `ssh-copy-id` 式的手法把 terminfo 传过去：

```console
$ infocmp -x | ssh remote -- tic -x -
```

**五、CSI u 必须在远端也开。** [第三篇](03-terminal-keys.html) 讲过，扩展编码要逐层
贯通。远端 tmux 如果没开 `extended-keys on`，你精心设计的 `Ctrl+Shift+*` 组合会
静默降级。

### 环境变量陷阱

还有一个更隐蔽的坑，我单独记过一份笔记。tmux 是长期运行的 server，**新建 pane 时
继承的是 tmux session 保存的环境，而不是当前终端的实时环境**。

而 `update-environment` 有个反直觉的行为：如果当前 client 环境里**没有**某个变量，
tmux 会把 session 里的这个变量标记为 **removed**。

于是这个序列会出问题：

```text
1. 本地图形终端 attach tmux session   → DISPLAY 正常
2. 从 macOS ssh 过来，tmux a          → SSH shell 没有 DISPLAY
                                       → tmux 把 session 的 DISPLAY 标记为移除
3. 回到本地图形终端，新开 pane        → 继承到"被移除"的环境，没有 DISPLAY
4. 输入法自动切换全部失效
```

诊断：

```console
$ tmux show-environment DISPLAY
-DISPLAY              ← 减号前缀 = 已被标记移除
```

规避办法（按推荐程度）：

```console
# 一、ssh attach 时用 -E 跳过 update-environment
$ tmux attach -E

# 二、更彻底：本地和 ssh 用不同的 tmux socket
$ tmux new -A -s work            # 本地
$ tmux -L ssh new -A -s ssh      # ssh

# 三、补全需要刷新的变量（不能单独解决 ssh 清空问题）
set-option -ga update-environment " DBUS_SESSION_BUS_ADDRESS XDG_RUNTIME_DIR WAYLAND_DISPLAY"
```

我的 `.tmux.conf` 里目前只补了 `PATH`：

```tmux
# Refresh PATH from client environment when creating new sessions,
# so tools like nvm/pyenv won't get stale paths cached by tmux server.
set-option -ga update-environment " PATH"
```

### ssh 自己的转义字符

最后一个小知识：ssh 客户端也吃键。默认的 escape character 是 `~`，**且只在行首
生效**：

| 序列 | 作用 |
| --- | --- |
| `~.` | 断开连接（连接卡死时的救命稻草） |
| `~^Z` | 把 ssh 挂起到后台 |
| `~#` | 列出转发的连接 |
| `~?` | 帮助 |

`ssh -e none` 可以关掉它。嵌套 ssh 时要按多个 `~`（`~~.` 断开第二层）。

## 设计原则：让分层可预测

诊断技巧只能救火。**真正的解法是让冲突从一开始就不会发生。**

### 原则一：一层一个修饰键

给每一层分配专属的修饰键或前缀，不要共用：

| 层 | 专属标识 | 理由 |
| --- | --- | --- |
| 窗口管理器 | `Mod1`（物理 Win 键） | 物理位置独立，终端程序几乎不用 |
| 终端 | `Ctrl+Shift+*` | 传统编码下不存在，零冲突 |
| tmux | `Alt+b` 前缀 | 单一入口，后续键不占用命名空间 |
| 终端内程序 | `Ctrl+*` | 物理 Alt 键，符合 readline/vim 传统 |
| Neovim 功能 | `,` leader | 前缀模式，可无限扩展 |

**关键是"专属"这两个字。** 一旦某个修饰键被两层共用，你就必须逐个记住哪些组合归谁，
这正是"状态确认"成本。

### 原则二：前缀 / leader 优于组合键

前缀模式（tmux 的 prefix、Neovim 的 leader）有两个组合键没有的优点：

1. **命名空间隔离**：`,` 后面的所有键都归 Neovim，不会跟任何层冲突。
2. **可发现性**：按下前缀之后可以弹出提示（which-key、tmux 的 `list-keys`），
   不需要背。

代价是多按一个键。但按照系列主线的标准——**多按一个键的成本，远低于"我不确定这个
组合归谁"的认知成本**。

### 原则三：越低层，占用越少

```text
① keyd        只改键的身份，不占任何"功能"
② WM          只抢必须无条件生效的（切窗口、启动终端、音量）
④ 终端        只抢终端自身的（复制粘贴、tab）
⑥ tmux        全部收进 prefix 之后
⑦ 应用        自由使用剩下的全部空间
```

理由很简单：**低层抢走的键，上层永远拿不回来**。上层抢走的键，你换个程序就自由了。
所以越低层越要克制。

第七到十一篇会逐层展开这条原则的具体实践。

## 小结

1. **责任链是 ① keyd → ② WM → ③ 输入法 → ④ 终端 → ⑤ termios → ⑥ tmux → ⑦ 应用**，
   每层都能吃掉按键。
2. **输入法夹在 WM 和终端之间**，所以 WM 全局键不受输入法影响，终端及以下全受影响。
3. **诊断从下往上二分，`cat -v` 是分水岭**：它能一刀切开图形层问题和终端层问题。
4. **合成按键（xdotool/XTEST）会从 ② 层重走整条链**，可能触发全局快捷键。
5. **ssh 让责任链串联两套**，①②③④ 全在本地；tmux 环境变量会被 ssh attach 污染。
6. **预防优于诊断**：一层一个专属修饰键，前缀优于组合键，越低层越克制。

下一篇从最上层开始，逐层看具体配置：
[窗口层：AwesomeWM 的克制](07-awesome.html)。
