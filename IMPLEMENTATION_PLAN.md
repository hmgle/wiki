# Linux 按键体系系列文章 — 实施计划

## 背景

把散落在 `~/code-x/dotconfig`（wezterm / tmux / zsh / awesome / rime / xmodmap /
udev）、`~/.config/nvim`、`~/.local/bin/keymap_ng.sh` 中的按键配置经验，梳理成一套
有主线、可独立成篇、层层递进的技术文章系列，发布到 wiki 的 `content/keymap/`。

**系列评价标准**：不把"效率"等同于少按几个键，而是以
**保护注意力、减少状态确认、降低错误恢复成本** 作为更高层标准。

**四个核心问题**：

1. 按下一个键，从物理层到应用层到底发生了什么？（篇 1–3）
2. ssh / awesome / wezterm / tmux / zsh / nvim 组合下，一个键被谁捕获、按什么顺序处理？（篇 6，铺垫于 4–5）
3. zsh / wezterm / nvim 三处"自动切换 rime 中英文"如何实现、底层原理是什么？（篇 12）
4. 怎样设计不迫使反复确认状态、不中断心流、可长期维护的键位系统？（篇 7–11 分层示范，篇 13 收束）

## 站点约定（已确认）

- 生成器：simiki3；构建 `make gen`，预览 `make preview`，CI 推 `master` 自动部署。
- 文章路径：`content/<category>/<name>.md`，本系列用 `content/keymap/`。
- Front matter：YAML，字段 `title` / `layout: page` / `date` / `updated`；正文首行 `[TOC]`。
- 语言：中文；代码块标注语言；正文用 `##` 起始层级。

## 素材索引（已完成勘察）

| 主题 | 来源 |
| --- | --- |
| xmodmap 三层模型实战 | `dotconfig/home/.Xmodmap`、`.pokerXmodmap`、`.tpXmodmap`、`bin/keymap.sh`、`~/.local/bin/keymap_ng.sh` |
| 热插拔 | `dotconfig/system/udev/80-keyboard.rules`、`bin/pokerkeyboard_{in,out}.sh`、`home/.xinitrc` |
| 窗口层 | `dotconfig/config/awesome/rc.lua`（modkey=Mod1，globalkeys/clientkeys/tag keys） |
| 终端层 | `dotconfig/config/wezterm/{wezterm,keybinds,utils}.lua`（leader、csi_u、SendKey 回调） |
| 复用层 | `dotconfig/home/.tmux.conf`（prefix M-b、extended-keys csi-u、escape-time 10） |
| 命令行层 | `dotconfig/home/.zshrc`（`add-zle-hook-widget line-init`、`bindkey`） |
| 编辑层 | `~/.config/nvim/{lua/basic.lua,lua/options.lua,viml/conf.vim,lua/plugins.lua}`、README |
| 中英文联动 | `dotconfig/config/wezterm/docs/rime-tmux-prefix.md`、`nvim/docs/im-switching.md`、`nvim/docs/input-method-switching.md`、`dotconfig/docs/tmux-environment-trap.md` |
| 输入法 | `~/.config/fcitx5/{config,profile}`、`dotconfig/rime/default.custom.yaml`、`rime/README.md` |

## 阶段划分

### Stage 1: 骨架与地基篇

**Goal**: 建立 `content/keymap/`，产出系列总览 + 篇 1（原理打底）。
**Success Criteria**: `make gen` 无报错；输出 HTML 含 TOC 与代码高亮。
**Status**: Complete

- `00-overview.md` — 系列总览、主线、评价标准、目录
- `01-key-journey.md` — 一次按键的旅程：从键盘矩阵到应用程序

### Stage 2: 编码与翻译层

**Goal**: 讲清"同一个键在每一层叫什么名字"，以及终端为何限制键位。
**Success Criteria**: 篇 2、3 各自可独立阅读，且被后续篇章引用。
**Status**: Complete

- `02-key-encoding.md` — 四层编码：scancode / keycode / keysym / 字符
- `03-terminal-keys.md` — 终端里的按键：转义序列、terminfo 与 CSI u

### Stage 3: 重塑键盘（系统层实战）

**Goal**: xmodmap 三层模型 + keyd/udev，给出可复现实战。
**Success Criteria**: 完整解释用户现有 `.Xmodmap` 每一行的作用与顺序陷阱。
**Status**: Complete

- `04-xmodmap.md` — X11 三层模型：keycode / keysym / modifier 组
- `05-keyd.md` — 到内核层重塑键盘：keyd、udev 与热插拔

### Stage 4: 分层捕获与各层配置

**Goal**: 回答核心问题 2，并逐层示范键位设计。
**Success Criteria**: 篇 6 给出可操作的冲突二分诊断流程。
**Status**: Complete

- `06-who-gets-the-key.md` — 谁先拿到这个键：捕获顺序与冲突诊断（含 ssh）
- `07-awesome.md` — 窗口层：AwesomeWM 的克制
- `08-wezterm.md` — 终端层：WezTerm
- `09-tmux.md` — 复用层：tmux
- `10-zsh-zle.md` — 命令行层：zsh ZLE
- `11-neovim.md` — 编辑层：Neovim

### Stage 5: 输入法联动与收束

**Goal**: 回答核心问题 3、4。
**Success Criteria**: 三处 rime 切换的原理、竞态与边界讲透；末篇给出可长期维护的设计原则。
**Status**: Complete

- `12-rime-auto-switch.md` — 中英文自动切换：fcitx5 + rime 三处联动
- `13-philosophy.md` — fcitx5 极简配置与键位设计哲学

## 完成后

全部阶段完成后删除本文件。
