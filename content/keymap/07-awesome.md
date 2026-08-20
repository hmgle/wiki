---
title: "Linux 按键体系（七）：窗口层的克制"
layout: page
date: 2026-08-20
updated: 2026-08-20
---

[TOC]

> 本文是 [Linux 按键体系系列](00-overview.html) 的第七篇。
> [上一篇](06-who-gets-the-key.html) 给出了责任链和"越低层越克制"的原则，本篇看
> 责任链上第一个真正会"吃键"的层：窗口管理器。

## 全局键的代价

窗口管理器通过 `XGrabKey` 注册全局快捷键。这个机制有一个不可协商的性质：

> **被 grab 的组合，任何应用程序都收不到，而且没有例外机制。**

不像 Web 页面里 `preventDefault` 之前还能让 handler 决定放不放行，X11 的 grab 是
静态的：注册了就永远抢，除非窗口管理器主动注销。

所以窗口层的每一个绑定都是一次**永久性征用**。判断标准应该是：

> 这个操作是不是**无论焦点在哪个程序、无论那个程序在做什么，都必须立即生效**？

符合的：切换窗口、切换工作区、启动终端、调音量、锁屏、截图。
不符合的：复制粘贴、搜索、保存——这些每个程序都有自己的语义。

AwesomeWM 4.3 的默认配置在这方面还算克制，我在它基础上又收敛了一些。

## modkey 的选择

```lua
-- config/awesome/rc.lua:197
modkey = "Mod1"
```

`Mod1` 通常是 Alt。但回忆 [第四篇](04-xmodmap.html) 那个交换：

```
keycode 64 = Super_L      ! 物理左 Alt  → Super_L
keycode 133 = Alt_L       ! 物理 Win 键 → Alt_L
add control = Super_L     ! 物理左 Alt  → Ctrl 修饰键
add mod1 = Alt_L          ! 物理 Win 键 → mod1
```

所以 `Mod1` 在我这里落在**物理 Windows 键**上。

这个选择的三条理由：

**一、物理隔离。** 物理 Windows 键在终端环境里几乎无人使用——readline、Vim、tmux
的传统键位全部建立在 Ctrl 和 Alt 上。把 WM 放在一个"没人抢"的物理键上，等于从源头
消除了冲突。

**二、拇指资源留给高频操作。** 拇指底下那个键（键帽印 Alt）给了 Ctrl，因为 Ctrl 在
终端里的使用频率高一个数量级。窗口切换再频繁，也远不如 `Ctrl+a/e/w/r` 频繁。

**三、`Mod1` 而不是 `Mod4` 是历史包袱。** awesome 默认用 `Mod4`（Super）。我用
`Mod1` 是因为多年前先做了物理键交换，配置就一直沿用下来了。

第三条是诚实的：**如果今天从零开始，直接用 `Mod4` + 不做物理交换，能达到同样效果
且少一层间接。** 但已有的肌肉记忆迁移成本超过了这点收益——这本身也是键位设计里
一个真实的约束，[第十三篇](13-philosophy.html) 会展开。

## globalkeys 与 clientkeys

awesome 把键位分成两张表，区别很重要：

```lua
-- globalkeys：全局，无论有没有窗口获得焦点都生效
globalkeys = gears.table.join(
    awful.key({ modkey }, "Return", function() awful.spawn(terminal) end, ...),
    ...
)
root.keys(globalkeys)

-- clientkeys：只在有窗口获得焦点时生效，作用于该窗口
clientkeys = gears.table.join(
    awful.key({ modkey }, "f", function(c) c.fullscreen = not c.fullscreen; c:raise() end, ...),
    ...
)
```

`clientkeys` 通过 `awful.rules` 挂到每个窗口上：

```lua
{ rule = { },
  properties = { ..., keys = clientkeys, ... }
},
```

**这个区分是"最小征用"原则的直接工具**：所有针对单个窗口的操作（全屏、关闭、最大化、
浮动）都应该放 `clientkeys`，因为它们在没有窗口时本来就没有意义。

## 完整键位表

按 awesome 自己的 `group` 分类。这张表本身就是本文的论据：

### launcher —— 启动器

| 组合 | 动作 |
| --- | --- |
| `Mod+Return` | 打开终端 |
| `Mod+space` | `rofi -show drun`（启动程序） |
| `Mod+grave` | `rofi -show window`（切换窗口） |
| `Mod+d` | 文件管理器 |
| `Mod+r` | awesome 内置 run prompt |
| `Mod+p` | menubar |

注意 `Mod+grave`：因为 [第四篇](04-xmodmap.html) 的三键轮换，**物理 Esc 键现在产生
`grave`**。所以这个绑定实际是 `Mod + 物理 Esc 键` → 窗口切换器。左上角那个键在
Vim 环境里已经没用了，正好拿来做全局窗口跳转。

### client —— 窗口操作

| 组合 | 动作 | 表 |
| --- | --- | --- |
| `Mod+j` / `Mod+k` | 焦点移到下/上一个窗口 | global |
| `Mod+Shift+j` / `Mod+Shift+k` | 与下/上一个窗口交换位置 | global |
| `Mod+Tab` | 回到上一个焦点窗口 | global |
| `Mod+u` | 跳到有 urgent 标记的窗口 | global |
| `Mod+Ctrl+n` | 恢复最小化的窗口 | global |
| `Mod+f` | 全屏 | client |
| `Mod+m` | 最大化 | client |
| `Mod+Ctrl+m` / `Mod+Shift+m` | 垂直/水平最大化 | client |
| `Mod+n` | 最小化 | client |
| `Mod+t` | 置顶 | client |
| `Mod+o` | 移到另一个屏幕 | client |
| `Mod+Ctrl+space` | 切换浮动 | client |
| `Mod+Ctrl+Return` | 与 master 交换 | client |
| `Mod+Shift+c` | 关闭窗口 | client |

`Mod+Shift+c` 关窗口——**破坏性操作加了 Shift**。这是一条应该贯穿所有层的规则：
不可撤销的操作不给单键。对比 `Mod+n`（最小化，可恢复）就是单键。

### tag —— 工作区

| 组合 | 动作 |
| --- | --- |
| `Ctrl+1`…`Ctrl+9` | 切换到第 N 个 tag |
| `Mod+Left` / `Mod+Right` | 上/下一个 tag |
| `Mod+Escape` | 回到上一个 tag |
| `Mod+Ctrl+N` | 叠加显示第 N 个 tag |
| `Mod+Shift+N` | 把当前窗口移到第 N 个 tag |
| `Mod+Ctrl+Shift+N` | 让当前窗口同时出现在第 N 个 tag |

```lua
awful.key({ "Control" }, "#" .. i + 9, function()
    local screen = awful.screen.focused()
    local tag = screen.tags[i]
    if tag then tag:view_only() end
end, { description = "view tag #" .. i, group = "tag" }),
```

**注意 `"#" .. i + 9` 这个写法**——它直接用 X keycode 而不是 keysym。回忆
[第二篇](02-key-encoding.html)：数字键 `1` 的 X keycode 是 10，所以 `i=1` 时是
`#10`。用 keycode 的好处是**不受键盘布局影响**：无论用 QWERTY 还是 Dvorak，
"顶排第一个键"永远是 keycode 10。awesome 默认配置的注释说得很清楚：

```lua
-- Be careful: we use keycodes to make it work on any keyboard layout.
```

再注意修饰键：**这里用的是 `Control`，也就是物理 Alt 键**。这是全套配置里最高频的
操作（切工作区），放在了拇指位置。代价见 [上一篇](06-who-gets-the-key.html)：
`Ctrl+2/6/7/8` 这几个 ASCII 控制字符在终端里拿不到了。

### layout —— 布局调整

| 组合 | 动作 |
| --- | --- |
| `Mod+h` / `Mod+l` | 减小/增大 master 区域宽度 |
| `Mod+Shift+h` / `Mod+Shift+y` | 增加/减少 master 窗口数 |
| `Mod+Ctrl+h` / `Mod+Ctrl+l` | 增加/减少列数 |
| `Mod+,` / `Mod+.` | 上/下一个布局 |

这里有一处**真实的不一致**，值得记下来：`Mod+Shift+h` / `Mod+Shift+y` 这一对。
按 hjkl 的语义应该是 `h`/`l` 配对，但 `Mod+Shift+l` 已经被"关屏幕"占用了：

```lua
awful.key({ modkey, "Shift" }, "l", function()
    awful.spawn("bash -c 'sleep 0.5 && xset dpms force off'")
end, { description = "turn off screen", group = "system" })
```

于是"减少 master 窗口数"被挤到了 `y` 上。这是配置演化中很典型的痕迹：**一个后加的
绑定挤掉了原有的对称性**。

它的代价不是"多记一个键"，而是**破坏了可预测性**——你原本可以从"h 是减、l 是增"
推导出所有绑定，现在这条规则有了例外，你必须记住例外。按系列主线的标准，这比多按
一个键糟糕得多。

诚实的评价：这个绑定应该改。`Mod+Shift+l` 让给 nmaster，关屏幕换一个不在
hjkl 语义空间里的键。

### system 与 awesome

| 组合 | 动作 |
| --- | --- |
| `Print` | 延时 9 秒截图 |
| `Mod+Shift+s` | 框选截图 |
| `Mod+Ctrl+s` | 延时截图到剪贴板 |
| `Mod+Ctrl+Shift+s` | 框选截图到剪贴板 |
| `Mod+Shift+l` | 关闭屏幕 |
| `Mod+s` | **显示快捷键帮助** |
| `Mod+Ctrl+r` | 重启 awesome |
| `Mod+Shift+q` | 退出 awesome |

截图那四个是一组正交设计：**`Shift` = 框选，`Ctrl` = 进剪贴板**，两个维度自由组合。
四个功能只需要记两条规则。这跟上面 `Shift+h`/`Shift+y` 的例外形成了鲜明对比——
**同一份配置里，好的部分和坏的部分并存，这很正常。**

## 可发现性：`Mod+s`

```lua
awful.key({ modkey }, "s", hotkeys_popup.show_help,
          { description = "show help", group = "awesome" }),
```

按 `Mod+s` 弹出所有快捷键的分组列表。这个功能能工作，是因为**每一条绑定都带了
`description` 和 `group` 元数据**：

```lua
awful.key({ modkey }, "j", function()
    awful.client.focus.byidx(1)
end, { description = "focus next by index", group = "client" }),
```

这是一个被严重低估的设计：**配置文件自己就是文档，而且是可执行的文档**。

对照系列主线的第二条标准——**减少状态确认**。忘记某个键位时，如果你要去翻配置文件，
那是一次上下文切换（离开当前工作、打开编辑器、搜索、理解、切回来）。如果按一下
`Mod+s` 就能看到，那只是一次瞬间查询。

**代价是写配置时多打几个字。收益是三个月后你还记得这套键位。** 这笔交易在任何有
超过 20 个绑定的配置里都是划算的。第十一篇讲 Neovim 的 which-key 时会再回到这个
主题。

## 方向语义的一致性

把所有跟"方向"有关的绑定放在一起看：

| 组合 | 语义 |
| --- | --- |
| `Mod+j` / `Mod+k` | 焦点 下/上 |
| `Mod+Shift+j` / `Mod+Shift+k` | 移动窗口 下/上 |
| `Mod+Ctrl+j` / `Mod+Ctrl+k` | 屏幕 下/上 |
| `Mod+h` / `Mod+l` | 尺寸 减/增 |
| `Mod+Ctrl+h` / `Mod+Ctrl+l` | 列数 增/减 |

规律是：

```text
j/k  = 纵向移动（焦点、窗口、屏幕）
h/l  = 横向调整（宽度、列数）
Shift = "带着当前窗口一起"
Ctrl  = "换一个容器层级"（屏幕、列）
```

**这套规则的价值在于可推导。** 你不需要记住 10 个绑定，只需要记住 4 条规则，剩下的
可以现推。而且这套 `hjkl` 语义在 tmux（`prefix + hjkl` 切 pane）和 Neovim
（`Ctrl+hjkl` 切窗口）里是一致的——**跨层的一致性比层内的完备性更有价值**。

不过要注意 `Mod+Ctrl+h`/`Mod+Ctrl+l` 是**增/减列数**，方向感跟 `Mod+h`/`Mod+l` 相反
（`h` 是增不是减）。这是 awesome 默认配置就有的，也是一处应该修的不一致。

## 该放窗口层的和不该放的

总结成一张判断表：

| 放窗口层 | 理由 |
| --- | --- |
| 切换窗口 / 工作区 | 必须无条件生效，且没有应用程序会用 |
| 启动终端 / 启动器 | 当前程序卡死时也要能开新窗口 |
| 音量 / 亮度 / 锁屏 | 硬件级操作，全局语义唯一 |
| 截图 | 需要在任何程序上方工作 |

| 不放窗口层 | 理由 |
| --- | --- |
| 复制 / 粘贴 | 每个程序语义不同（终端的选区、编辑器的寄存器） |
| 搜索 / 保存 | 同上 |
| 分屏 | tmux 和终端各有自己的分屏，抢了会打架 |
| 输入法切换 | 见 [第十二篇](12-rime-auto-switch.html)，应该自动化而非手动 |

最后一行值得展开：很多人会在 WM 层绑一个"切换输入法"的全局键。按系列主线的标准
这是个坏设计——**它把状态管理的责任推给了用户**。正确的方向是让输入法状态在合适的
时机自动回到正确的值，用户根本不需要这个键。

## 小结

1. **`XGrabKey` 是永久征用**，没有例外机制，所以窗口层每个绑定都要问："这是不是
   必须无条件生效？"
2. **modkey 放在物理上没人抢的键**（Windows 键），从源头消除跨层冲突。
3. **`globalkeys` / `clientkeys` 的区分是最小征用的工具**，窗口相关操作全放后者。
4. **每条绑定都写 `description` + `group`**，配置即文档，`Mod+s` 可查——这是
   "减少状态确认"最便宜的实现。
5. **方向语义要跨层一致**（`hjkl` 在 WM/tmux/Neovim 里含义相同），可推导 > 可记忆。
6. **不一致会传染。** `Mod+Shift+l` 被占用导致 `Mod+Shift+y` 这种例外，代价不是多记
   一个键，而是整套规则不再可推导。

下一篇看终端层：
[终端层：WezTerm](08-wezterm.html)。
