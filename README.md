# hmgle Wiki

这是 hmgle 的个人技术 Wiki，使用 [Simiki3](https://github.com/hmgle/simiki3) 将 Markdown 文章生成静态 HTML 页面。

在线访问：<https://hmgle.github.io/wiki/>

## 目录结构

```text
.
├── .github/workflows/pages.yml  # GitHub Actions 构建与部署流程
├── _config.yml                  # Wiki 配置
├── content/                     # Markdown 文章源文件
│   ├── search/                  # 搜索相关笔记
│   ├── tip/                     # 技巧类文章
│   └── tool/                    # 工具类文章
├── themes/simple2/              # Wiki 主题
└── output/                      # 本地构建生成的静态站点（不提交）
```

文章通常放在 `content/<category>/` 下，并使用 YAML front matter 设置标题、日期和布局，例如：

```markdown
---
title: "文章标题"
layout: page
date: 2026-08-02
updated: 2026-08-02
---

文章内容。
```

## 本地构建

本仓库使用 [uv](https://docs.astral.sh/uv/) 临时运行 simiki3，不要求全局安装旧版 Simiki。当前 CI 固定使用 simiki3 的一个 Git commit，以保证构建结果稳定：

```bash
SIMIKI3_REQUIREMENT="simiki3 @ git+https://github.com/hmgle/simiki3.git@b8a1100473e9bd3aafb3e402ef923d45ca93cf24"

uv run --isolated --with "$SIMIKI3_REQUIREMENT" simiki3 --version
uv run --isolated --with "$SIMIKI3_REQUIREMENT" simiki3 theme sync simple2 --path .
uv run --isolated --with "$SIMIKI3_REQUIREMENT" simiki3 build .
```

构建完成后，静态文件位于 `output/`。也可以启动本地预览服务器，并在修改文章后自动重新构建：

```bash
uv run --isolated --with "$SIMIKI3_REQUIREMENT" simiki3 serve . --watch
```

## GitHub Actions 部署

`.github/workflows/pages.yml` 在以下情况下运行：

- 推送到 `master`：构建并部署到 GitHub Pages；
- 针对 `master` 创建或更新 Pull Request：只构建并检查，不部署；
- 手动执行 workflow：构建并部署。

部署流程会检查关键输出文件，包括首页、Atom feed、主题 CSS、搜索页和示例文章，然后通过 GitHub Pages artifact 发布。新增或修改 `content/` 下的文章后，推送到 `master` 即可触发自动部署。

## 新增文章

可以直接在对应分类目录下创建 Markdown 文件，也可以使用 simiki3 的脚手架命令：

```bash
uv run --isolated --with "$SIMIKI3_REQUIREMENT" \
  simiki3 new "文章标题" --path . --category tip
```

提交前建议先执行一次本地构建，确认生成页面和链接正常，再提交 Markdown 源文件。`output/` 属于生成目录，不应手动编辑或提交。
