# Agent 说明：XeriChen.github.io

本文件给 AI coding agent（Grok Build / Claude Code / Cursor 等）使用，说明如何安全、正确地维护这个个人博客仓库。人类读者可先看 `README.md`。

## 1. 项目是什么

- **类型：** 个人静态博客
- **站点：** https://xerichen.github.io
- **仓库：** https://github.com/XeriChen/XeriChen.github.io
- **生成器：** [Hugo](https://gohugo.io) **Extended**
- **主题：** [Blowfish](https://blowfish.page/)（Git **submodule**，路径 `themes/blowfish`）
- **部署：** GitHub Actions → GitHub Pages（`.github/workflows/hugo.yaml`）
- **语言：** 默认内容语言 `zh-cn`（简体中文）
- **内容管理（可选）：** Pages CMS，配置见 `.pages.yml`

本地未安装 Hugo 时，不要假装已构建成功；可只改源码，由 CI 构建。若需本地预览，使用 **Hugo Extended**，版本需落在主题声明区间内（见下节）。

## 2. 工具链版本（以仓库为准）

| 组件 | 当前约定 | 说明 |
|------|----------|------|
| Hugo（CI） | `0.163.3` extended | `.github/workflows/hugo.yaml` 中 `HUGO_VERSION` |
| Blowfish | 子模块 pin 的 tag/commit | 勿只改工作区而不更新 gitlink |
| 主题声明的 Hugo | min `0.158.0`，max `0.163.3` | 见 `themes/blowfish/config.toml` 的 `[module.hugoVersion]` |

升级主题后**必须**再读主题 `config.toml` 的 `min`/`max`，并同步改 CI 的 `HUGO_VERSION`。不要升到超过主题 `max` 的 Hugo，除非已确认主题兼容。

克隆与子模块：

```bash
git clone --recurse-submodules git@github.com:XeriChen/XeriChen.github.io.git
# 若已克隆但 themes/blowfish 为空：
git submodule update --init --recursive
```

## 3. 目录结构（agent 该动哪里）

```
.
├── AGENTS.md                 # 本文件（agent 规则）
├── README.md                 # 人类向简介
├── .github/workflows/hugo.yaml   # 构建与部署
├── .gitmodules               # blowfish 子模块定义
├── .pages.yml                # Pages CMS 字段
├── archetypes/               # hugo new 模板
├── config/_default/          # 站点配置（主配置区）
│   ├── hugo.toml
│   ├── languages.zh-cn.toml  # 语言 / 作者 / locale
│   ├── menus.zh-cn.toml      # 导航菜单与标签入口
│   ├── markup.toml           # Goldmark / KaTeX 分隔符
│   ├── params.toml           # Blowfish 主题参数
│   └── module.toml           # 当前为空；未用 Hugo Modules
├── content/posts/            # 博客正文（主要工作区）
├── public/                   # 本地/历史构建产物（见第 7 节）
└── themes/blowfish/          # 主题 submodule，勿当普通目录乱改
```

**优先编辑：** `content/posts/`、`config/_default/`、`archetypes/`、本文件与 `README.md`。  
**谨慎编辑：** `.github/workflows/`、`.gitmodules`、主题子模块指针。  
**不要当主题源码仓库改：** `themes/blowfish/**` 内业务定制应尽量用站点侧 `config` / 覆盖布局（若以后引入 layouts），而不是直接 fork 式改 submodule 内文件（除非用户明确要求升级或 patch 主题）。

## 4. 文章规范

### 4.1 新建

优先：

```bash
hugo new posts/your-slug.md
```

无 Hugo 时可手写 `content/posts/<slug>.md`，front matter 与 archetype 对齐。

### 4.2 Front matter（YAML）

与 `archetypes/default.md`、`.pages.yml` 一致，推荐字段：

```yaml
---
title: "文章标题"
date: 2026-07-19T13:57:43+08:00
draft: false
tags: ["学习笔记"]
description: "一句话摘要，用于列表/SEO，可为空字符串"
---
```

约定：

- `date` 使用带时区的 ISO 时间，时区 **`+08:00`**（与 CI `TZ: Asia/Shanghai` 一致）。
- `draft: true` 不会出现在生产构建（CI 未开 `buildDrafts`）。
- `tags` 为字符串列表。现有导航里已挂过的标签入口包括（见 `menus.zh-cn.toml`）：`mbti`、`love`、`travel` 等；新标签可不进菜单，但会出现在标签页。
- 正文以 **简体中文** 为主；技术术语可保留英文。
- 文件名可用英文 slug（如 `grok-build-guide.md`）或中文（如 `遵义游玩攻略-景点篇.md`）；新建时优先 **简短英文/拼音 slug**，减少编码与链接问题。
- 数学公式：站点已开 passthrough，可用 `$$...$$` 或 `\[...\]`；部分旧文使用 `{{</* katex */>}}` shortcode，保持与同文一致即可。
- 图片：现有文章多用外链图床（如 GitHub raw）。新增图片时沿用用户习惯；不要擅自引入未说明的大体积二进制，除非用户要求。

### 4.3 常用内容操作清单

| 任务 | 做法 |
|------|------|
| 写新文章 | 新增 `content/posts/*.md` + front matter |
| 改站名/作者/简介 | `config/_default/languages.zh-cn.toml` |
| 改主题外观/首页 | `config/_default/params.toml` |
| 改顶栏菜单 | `config/_default/menus.zh-cn.toml` |
| 改分页/站点级 Hugo 选项 | `config/_default/hugo.toml` |
| 改 CMS 表单字段 | `.pages.yml` |

语言配置须使用 Hugo 0.158+ 字段：`locale`、`label`（不要写已弃用的 `languageCode` / `languageName`）。

## 5. 构建与预览

```bash
# 生产风格构建（与 CI 接近）
hugo --gc --minify

# 本地预览（含草稿）
hugo server -D
```

CI 构建要点（见 workflow）：

- `actions/checkout`：`submodules: recursive`，`fetch-depth: 0`
- Hugo Extended `HUGO_VERSION`
- `hugo --gc --minify --baseURL "<pages base URL>/"`
- 产物目录 `public`，上传为 Pages artifact 并部署

推送到 **`main`** 会触发部署；`workflow_dispatch` 可手动跑。

## 6. 主题升级流程（Blowfish）

1. 确认当前 pin：`git submodule status` / `git ls-tree HEAD themes/blowfish`
2. 更新子模块到目标 tag（例）：
   ```bash
   git submodule update --init --recursive
   cd themes/blowfish && git fetch --tags && git checkout vX.Y.Z
   cd ../..
   ```
3. 读取 `themes/blowfish/config.toml` 中 `[module.hugoVersion]`，更新 `.github/workflows/hugo.yaml` 的 `HUGO_VERSION`（保持 extended，且 ≤ max）
4. 对照 [Blowfish 配置文档](https://blowfish.page/docs/configuration/) 与 release notes，必要时迁移 `config/_default/*.toml`（如语言字段、弃用参数）
5. 本地能跑则 `hugo` 验证；不能则靠 CI
6. 提交时包含：**子模块 gitlink** + workflow + 任何配置迁移；commit 信息说明版本号

子模块远程：`git@github.com:nunocoracao/blowfish.git`，`.gitmodules` 中 `branch = main`。

## 7. 关于 `public/`

`public/` 为 Hugo 本地/CI 构建产物，已在 **`.gitignore`** 中忽略，**不要提交**。正式发布以 **GitHub Actions 现构建** 并上传 Pages artifact 为准。

Agent 默认行为：

- 日常改文章/配置：只提交 `content/`、`config/` 等源文件。
- 本地 `hugo` / `hugo server` 会生成 `public/`，保持未跟踪即可。
- 不要把 `public/` 从 `.gitignore` 拿掉并重新入库，除非用户明确要求改部署方式。

## 8. Git / PR 约定

- 默认分支：`main`；与 `origin/main` 同步后再推。
- 提交信息：完整句子，中文或英文均可；说明**为什么**，可带版本号（如「升级 Blowfish 至 v2.104.0」）。
- **仅在用户要求时** commit / push / 开 PR。
- 不要 `git commit --amend` 已推送提交，除非用户明确要求且满足安全条件。
- 不要强推 `main`，不要随意改 git config。
- 可用 `gh` 查看 Actions：`gh run list`、`gh run watch`。
- 子模块指针变更要进同一次提交，避免 CI checkout 到空主题。

## 9. 安全与边界

- 不提交密钥、token、`.env`、私人邮箱以外的敏感信息（站点作者邮箱已在配置中公开则除外）。
- 不运行破坏性命令（`rm -rf`、盘符清理、强制改历史等），除非用户明确要求且范围清楚。
- 不把用户未要求的重构、批量重命名标签、重写旧文风格当作「顺手优化」。
- 改菜单/标签结构前先读 `menus.zh-cn.toml` 与现有文章 tags，避免断链。
- 外链图片与攻略类内容可能含「待人工核实」说明；不要擅自改成「已核实」口吻。

## 10. 常见任务速查

**只发一篇文**

1. 添加 `content/posts/<slug>.md`
2. 设好 `title` / `date` / `draft` / `tags` / `description`
3. 用户要求时再 commit & push，观察 Actions 是否 green

**改站点展示名或社交链接**

- `config/_default/languages.zh-cn.toml` 的 `title`、`[params.author]`、`links`

**升级工具链**

1. 升 Blowfish submodule  
2. 对齐 Hugo CI 版本  
3. 修配置弃用项  
4. 推送并确认 Pages 构建成功  

**诊断 CI 失败**

1. `gh run list --workflow=hugo.yaml`  
2. 看是否 submodule 未初始化、Hugo 版本超主题 max、TOML 语法错误、front matter 坏掉  
3. 本地 `hugo --gc --minify` 复现（若已安装）

## 11. 参考链接

- Blowfish 文档：https://blowfish.page/docs/  
- Blowfish Releases：https://github.com/nunocoracao/blowfish/releases  
- Hugo Releases：https://github.com/gohugoio/hugo/releases  
- 主题要求（本仓库内）：`themes/blowfish/config.toml`  
- CI：`.github/workflows/hugo.yaml`

---

**原则：** 源码在 `content/` 与 `config/`；主题用 submodule 固定版本；部署交给 CI；少动 `public/`；版本变更成对升级（主题 + Hugo）；中文内容、上海时区、用户点头再推送。
