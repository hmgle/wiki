---
title: "Linux 按键体系 13：极简配置与键位哲学"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的最后一篇，回答系列的第四个核心
> 问题：**怎样设计一套不迫使用户反复确认状态、不中断心流，而且能长期维护的键位
> 系统？**
>
> 前十二篇拆开了每一层。这一篇把它们合起来。

## 先看一份极简的 fcitx5 配置

`~/.config/fcitx5/profile` 全文：

```ini
[Groups/0]
# Group Name
Name=Default
# Layout
Default Layout=us
# Default Input Method
DefaultIM=rime

[Groups/0/Items/0]
# Name
Name=rime

[GroupOrder]
0=Default
```

**一个组，一个输入法。** 没有拼音+五笔+英文的三件套，没有多个 group。

这不是"我只需要一种输入法"那么简单。它是一个刻意的架构决定：**输入法的数量直接
决定了状态空间的大小。**

```text
1 个输入法：状态空间 = {中文, 英文}              → 2 种
3 个输入法：状态空间 = {拼音, 五笔, 英文} × 中英  → 6 种
```

而"我现在在哪个状态"这个问题的认知成本，随状态数量增长。多装的那两个输入法，
90% 的时间不用，但 100% 的时间都在扩大你需要确认的状态空间。

### 热键：能关的全关掉

`~/.config/fcitx5/config` 的 Hotkey 段：

```ini
[Hotkey]
# Enumerate when press trigger key repeatedly
EnumerateWithTriggerKeys=True
# Temporally switch between first and current Input Method
AltTriggerKeys=
# Enumerate Input Method Forward
EnumerateForwardKeys=
# Enumerate Input Method Backward
EnumerateBackwardKeys=

[Hotkey/TriggerKeys]
0=Control+space
```

`AltTriggerKeys`、`EnumerateForwardKeys`、`EnumerateBackwardKeys` **全部清空**。

fcitx5 默认会占用一批全局热键（临时切换、正向轮换、反向轮换）。回忆
[第六篇](06-who-gets-the-key.html) 的责任链——输入法在第 ③ 层，它抢走的键，终端和
终端里的所有程序都收不到。

**只留一个 `Ctrl+space` 作为唯一入口。** 而且因为组里只有一个输入法，它的行为就是
纯粹的"开/关"，没有第三种可能。

### 默认英文

```ini
[Behavior]
# Active By Default
ActiveByDefault=False
```

新窗口默认英文态。

这跟 [第十二篇](12-rime-auto-switch.html) 的整套自动切换是同一个方向：**把"英文"
定义成默认状态，中文是需要显式进入的例外。** 因为在我的使用中，绝大多数按键是命令
而不是文字。

默认值应该是你 80% 时间需要的那个——这样 80% 的时间你不需要做任何事。

### 每个窗口独立记忆状态

```ini
# Reset state on Focus In
resetStateWhenFocusIn=No
# Share Input State
ShareInputState=No
```

`ShareInputState=No` 让**每个窗口保有自己的输入状态**；`resetStateWhenFocusIn=No`
让切回来时保持原样。

组合起来的效果是：浏览器地址栏是中文态、终端是英文态，来回切窗口时各自保持。
**空间位置就成了状态的记忆锚点**——你不需要记住状态，窗口帮你记着。

这跟 [第五篇](05-keyd.html) 说的"用空间关系代替符号记忆"是同一个思路，只是用在了
状态上。

### 不要打扰

```ini
# Show Input Method Information when switch input method
ShowInputMethodInformation=True
# Show Input Method Information when changing focus
showInputMethodInformationWhenFocusIn=False
# Show compact input method information
CompactInputMethodInformation=True
```

**切换输入法时提示（你主动做的事，给个确认），切换窗口时不提示（你没做这件事，
不要弹）。**

这条区分很值得学。通知的判断标准不是"这个信息重不重要"，而是**"用户此刻是不是在
等这个信息"**。切窗口时弹输入法提示，信息是对的，但没人在等它——那就是噪音。

`CompactInputMethodInformation=True` 是同一个方向的补充：非弹不可的时候，尽量小。

### 密码框

```ini
# Allow input method in the password field
AllowInputMethodForPassword=False
# Show preedit text when typing password
ShowPreeditForPassword=False
```

密码框里不启用输入法。既是安全考虑（预编辑串可能被截屏、被记录），也是可用性考虑——
密码几乎不可能是中文，在那里弹候选窗口纯属添乱。

### 候选窗口

```ini
[Hotkey/PrevPage]
0=Up
[Hotkey/NextPage]
0=Down
[Hotkey/PrevCandidate]
0=Shift+Tab
[Hotkey/NextCandidate]
0=Tab

[Behavior]
# Default page size
DefaultPageSize=5
```

`Tab`/`Shift+Tab` 选候选、`↑`/`↓` 翻页、每页 5 个。

`DefaultPageSize=5` 而不是默认的更多：**候选越多，你花在"扫描候选列表"上的时间
越长**。5 个是一眼能扫完的数量。如果前 5 个里没有，多半是打错了或者该换个词，
翻页去找第 8 个候选的期望收益很低。

配合 [第八篇](08-wezterm.html) 的 `ime_preedit_rendering = "Builtin"`——预编辑串
显示在光标位置，候选窗口紧贴其下。**视线不需要移动。**

## 键位设计的二十条

把全系列的取舍提炼成可操作的原则。分四组。

### 一、结构：谁拥有哪些键

**1. 一层一个专属修饰键。**

```text
② 窗口管理器  Mod1（物理 Windows 键）
④ 终端        Ctrl+Shift+*
⑥ tmux        Alt+b 前缀
⑦ 终端内程序  Ctrl+*（物理 Alt 键）
⑦ Neovim 功能 , leader
```

一旦某个修饰键被两层共用，你就必须逐个记住哪些组合归谁。
（[第六篇](06-who-gets-the-key.html)）

**2. 越低层越克制。** 低层抢走的键上层永远拿不回来；上层抢走的键，换个程序就自由
了。所以窗口管理器只抢"必须无条件生效"的，终端只抢"传统编码下本来就不存在"的。
（[第六篇](06-who-gets-the-key.html)、[第七篇](07-awesome.html)）

**3. 前缀优于组合键。** 前缀提供命名空间隔离和可发现性，代价只是多按一个键。
tmux 的插件全部收在 prefix 之后，装 20 个也不影响 Neovim 能绑什么。
（[第九篇](09-tmux.html)）

**4. 低层管"是什么键"，高层管"做什么事"。** keyd 里只放 Caps→Ctrl 这类身份映射，
不放"启动终端"这类语义操作——后者需要上下文才能决定该不该让路。
（[第五篇](05-keyd.html)）

**5. 物理位置分配按频率。** 拇指底下给 Ctrl（终端里最高频），Windows 键给窗口管理
器（低频但必须全局）。物理分离让"这个键归谁"变成一个手上的动作而不是脑子里的判断。
（[第四篇](04-xmodmap.html)）

### 二、认知：怎样不用背

**6. 可推导优于可记忆。** 四条规则（`j/k` 纵向、`h/l` 横向、`Shift` 带着走、
`Ctrl` 换层级）能推出十个绑定，比记十个绑定容易一个数量级。
（[第七篇](07-awesome.html)）

**7. 跨层一致性优于层内完备性。** `hjkl` 在 AwesomeWM、tmux、Neovim 里语义相同；
`Ctrl+a/e/k` 在 zsh 和 Neovim 命令行里语义相同。**跨层一致消除的是"我现在在哪一层"
这个问题。**（[第十篇](10-zsh-zle.html)、[第十一篇](11-neovim.html)）

**8. 用空间关系编码，别用符号记忆。** 分层里的 `hjkl` + `uiop` 是空间连续的；四个
分散的 `Ctrl+a`/`Ctrl+e`/`Alt+v`/`Ctrl+v` 只能死记。（[第五篇](05-keyd.html)）

**9. 分组前缀把记忆量从 N 降到 log N。** `,f` 查找、`,s` 符号、`,c` 复制；小写常用
变体、大写扩展变体。加第十个绑定时你知道该往哪放。（[第十一篇](11-neovim.html)）

**10. 可发现性要内建。** 每条绑定写 `description`/`desc`，配上 which-key 或
`Mod+s`。**关键不是省时间，是不用离开当前上下文。**
（[第七篇](07-awesome.html)、[第十一篇](11-neovim.html)）

**11. 不一致会传染。** `Mod+Shift+l` 被占用导致 `Mod+Shift+y` 这个例外，代价不是多
记一个键，而是**整套规则不再可推导**——你必须记住"有例外"，然后每次都想一下这是不是
那个例外。（[第七篇](07-awesome.html)）

### 三、状态：怎样不用确认

**12. 状态自动恢复优于状态可见。** 让"我现在什么状态"这个问题不需要被问出来，
比让答案更容易看到更好。（[第十二篇](12-rime-auto-switch.html)）

**13. 动作做成幂等的设置，不要做成 toggle。** 这能省掉整类同步问题，让"尽力而为"的
架构成立。`SetAsciiMode b true` 调用 100 次和 1 次结果相同。
（[第十二篇](12-rime-auto-switch.html)）

**14. 重要状态一眼可辨，不重要的东西不要动。** 非活动 tmux pane 调暗，光标不闪烁，
动画降到 1fps。**周边视觉对运动极度敏感，任何周期性运动都在持续消耗注意力。**
（[第八篇](08-wezterm.html)、[第九篇](09-tmux.html)）

**15. 模态的收益随编辑规模增长，成本随切换频率增长。** 编辑几百行代码用模态划算；
输入一行命令不划算。真要在命令行用 vi 模式，必须补上光标形状指示——否则就是"逼迫
用户反复确认状态"的教科书反例。（[第十篇](10-zsh-zle.html)）

**16. 通知的判断标准是"用户此刻是不是在等这个信息"**，不是"这个信息重不重要"。
切换输入法时提示，切换窗口时不提示。

### 四、风险与维护

**17. 破坏性操作不给单键。** `Mod+Shift+c` 关窗口，`Mod+n` 最小化——前者不可撤销，
加 Shift；后者可恢复，单键。（[第七篇](07-awesome.html)）

**18. 静默降级比功能缺失更糟。** 功能缺失你立刻知道；静默降级（CSI u 没配好导致
`Ctrl+i` 变成 `Tab`）要花时间才能意识到。所以核心键位只用传统编码能表达的组合。
（[第三篇](03-terminal-keys.html)、[第十一篇](11-neovim.html)）

**19. 任何自动机制都要有关闭开关。** 自动化在 95% 的情况下帮忙，5% 的情况下碍事；
如果那 5% 无法关闭，用户对整个机制的信任就没了。
（[第十二篇](12-rime-auto-switch.html)）

**20. 让错误的输入产生正确的结果。** 全角 `：` 直接映射成 `:`，全角 `，` 直接当
leader。**错误恢复成本降到零，因为根本没有错误发生。** 这比"检测错误再提示用户
修正"高一个层次。（[第十一篇](11-neovim.html)）

## 可长期维护：四条工程实践

原则之外，还有一些纯粹工程性的做法。

### 单一数据源

WezTerm 从 `~/.ssh/config` 自动生成 ssh domain 列表，加一台机器只改一个地方
（[第八篇](08-wezterm.html)）。

**任何需要在两个地方同步的配置，早晚会不同步。** 而不同步的配置比没有配置更糟——
你会基于其中一份做判断，然后被另一份坑到。

反例就在本系列里：`timeoutlen` 在 `viml/conf.vim` 里是 500，在 which-key 配置里是
300。今天你调 300 那个，明天可能改错成 500 那个然后困惑半天。

### 用框架的注册机制，不要覆盖

```text
zsh hook       add-zle-hook-widget line-init ✓    zle -N zle-line-init ✗
zsh precmd     add-zsh-hook precmd f          ✓    precmd() {...}       ✗
Neovim autocmd augroup + clear = true          ✓    裸 autocmd           ✗
tmux hook      set-hook -a                     ✓    set-hook             ✗
```

直接覆盖共享扩展点的后果是**静默破坏别人的功能**，而且故障现象跟你的改动毫无
表面关联——改输入法，坏方向键。（[第十篇](10-zsh-zle.html)）

### 改得越少越好

zsh 层我只加了一条 `bindkey`。**每一条自定义绑定都是一份需要记忆和维护的负债**，
也是一份跟"别人的机器"和"未来的自己"之间的差异。

加之前先确认默认键位真的不够用。

### 承认迁移成本

Neovim 的 leader 是 `,` 而不是空格，AwesomeWM 的 modkey 是 `Mod1` 而不是 `Mod4`。
两个都是"知道有更好选择但不改"的遗留决定。

**重要的是知道它是遗留选择，而不是把它合理化成设计。** 前者让你在有机会重来时做对；
后者让你把缺陷传给下一份配置。

## 评估一个新绑定

想加一个新绑定时，按顺序过这几个问题：

```text
1. 默认键位真的不够用吗？
   → 够用就别加

2. 它属于哪一层？
   → 用责任链定位（第六篇）：这个操作需要多大的作用域？
   → 能放高层就别放低层

3. 这一层的专属修饰键 / 前缀下还有位置吗？
   → Neovim: :KeyAnalyzer <前缀>
   → tmux:   tmux list-keys -T prefix
   → zsh:    bindkey
   → awesome: Mod+s

4. 它能被现有的分组规则推导出来吗？
   → 能 → 按规则放（如 ,f 下再加一个查找入口）
   → 不能 → 是不是该新开一个组？还是这个功能不该有键位？

5. 它会破坏跨层一致性吗？
   → hjkl 的方向语义、Ctrl+a/e 的行首行尾，这些不要动

6. 它需要 CSI u 吗？
   → 需要 → 只能用于非核心功能（会静默降级）

7. 它是破坏性的吗？
   → 是 → 加一个 Shift，或者放到 leader 之后

8. 三个月后我还能想起来吗？
   → 写 desc/description
   → 想不起来时有没有办法当场查到（which-key / Mod+s）
```

如果第 1 题就通不过，后面七题都不用问了——这是最常被跳过、也最有价值的一题。

## 定期审计

配置会腐化。写这个系列的过程本身就是一次审计，发现的问题列在这里，也算给这套配置
留一份 TODO：

| 问题 | 位置 | 篇 |
| --- | --- | --- |
| `Alt+1`–`Alt+6` 被 WezTerm 抢走，Neovim buffer 跳转只有 7–9 生效 | wezterm/keybinds.lua + nvim/plugins.lua | [11](11-neovim.html) |
| `Mod+Shift+h`/`Mod+Shift+y` 破坏 hjkl 对称（`Shift+l` 被关屏占用） | awesome/rc.lua | [7](07-awesome.html) |
| `Mod+Ctrl+h`/`Mod+Ctrl+l` 的增减方向与 `Mod+h`/`Mod+l` 相反 | awesome/rc.lua | [7](07-awesome.html) |
| `timeoutlen` 在两处设置（500 / 300） | viml/conf.vim + plugins.lua | [11](11-neovim.html) |
| `cnoremap <C-K> <C-U>` 是死代码（被后面的 `<C-k>` 覆盖） | viml/conf.vim | [11](11-neovim.html) |
| `.Xmodmap` 依赖初始状态、非幂等 | home/.Xmodmap | [4](04-xmodmap.html) |
| `guicursor=i:block` 放弃了光标指示模式 | viml/conf.vim | [11](11-neovim.html) |
| xmodmap + udev + 三份配置的热插拔补丁，应迁移到 keyd | dotconfig/ | [4](04-xmodmap.html)、[5](05-keyd.html) |

**这八条里有六条是单看任何一个文件都发现不了的**——它们是跨文件、跨仓库、跨层的
问题。这正是为什么需要一张全局的键位分配表，也是这个系列存在的理由。

建议的审计节奏：

- **加新绑定时**：跑一遍上面那个八题清单。
- **每半年**：对着责任链过一遍每层的键位表，找冲突和不一致。
- **换硬件 / 换终端 / 换发行版时**：必须重新审计，因为底层假设变了。

## 结语：什么是"好的键位系统"

回到系列开头那三条标准。

**保护注意力**不是"少按几个键"。手不离开主键区、光标不闪烁、非活动窗格调暗、
候选窗口贴着光标——这些都不省按键，但它们都在减少注意力被拉走的次数。

**减少状态确认**不是"把状态显示得更清楚"。最好的状态指示是不需要指示——输入法在
你按 Esc 时自动切回英文，新 pane 自动继承当前目录，你根本不需要问那个问题。

**降低错误恢复成本**不是"减少出错概率"。全角逗号直接当 leader 用，`Ctrl+[` 天然
被 Esc 的逻辑覆盖，未覆盖的路径按一次 Esc 就自愈——**让错误变成非错误，比让错误
变少更有效。**

这三条合起来指向同一件事：**好的键位系统，是让你能持续想着"我要做什么"，而不是
"我该怎么做到"。**

十三篇写完，从键盘矩阵的二极管一直讲到 D-Bus 调用。中间那些层——scancode、keycode、
keysym、转义序列、grab、prefix、widget——它们的存在本身就是历史的产物，不完美，
充满妥协。但正因为知道每一层在做什么，你才能判断一个问题该在哪一层解决，以及
**不该**在哪一层解决。

这大概就是这个系列最实际的价值：**不是给你一份配置，是给你一张地图。**

## 系列目录

- [00 系列总览](00-overview.html)
- [01 一次按键的旅程](01-key-journey.html)
- [02 四层编码](02-key-encoding.html)
- [03 终端里的按键](03-terminal-keys.html)
- [04 xmodmap 三层模型](04-xmodmap.html)
- [05 到内核层重塑键盘](05-keyd.html)
- [06 谁先拿到这个键](06-who-gets-the-key.html)
- [07 窗口层的克制](07-awesome.html)
- [08 终端层 WezTerm](08-wezterm.html)
- [09 复用层 tmux](09-tmux.html)
- [10 命令行层 zsh ZLE](10-zsh-zle.html)
- [11 编辑层 Neovim](11-neovim.html)
- [12 中英文自动切换](12-rime-auto-switch.html)
- 13 极简配置与键位哲学（本文）
