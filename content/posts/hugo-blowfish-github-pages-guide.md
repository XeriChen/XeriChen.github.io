---
title: "从零搭建 Hugo + Blowfish + GitHub Pages 博客指南"
date: 2026-07-29T15:30:00+08:00
draft: false
tags: ["学习笔记", "工具链", "Hugo", "Blowfish"]
description: "手把手教你用 Hugo Extended + Blowfish 主题 + GitHub Pages 搭一个静态博客，从环境配置、主题安装、自动部署到常见问题排查，附真实仓库配置参考。"
---

> **说明：** 这篇文章基于我搭自己博客时的实际步骤整理，所有配置项都以当前仓库代码为准。不会扯太多「为什么要用静态生成器」之类的理论，重点是**怎么做**以及**哪里容易踩坑**。
>
> 最终效果就是你现在看到的这个站点：<https://xerichen.github.io>
> 仓库地址：<https://github.com/XeriChen/XeriChen.github.io>

## 一、技术选型简述

| 组件 | 选型 | 理由 |
|------|------|------|
| 静态生成器 | **Hugo Extended** | 构建极快、单二进制部署、社区主题丰富 |
| 主题 | **Blowfish** | 现代感、暗色模式、可配置性强、文档齐全 |
| 托管 | **GitHub Pages** | 免费、与仓库原生集成、支持自定义域名 |
| CI/CD | **GitHub Actions** | 无需本地构建，推送即部署 |

> 为什么是 **Extended** 版本？因为 Blowfish 用到了 SCSS 编译，Hugo 的普通版不带这个功能，必须用 Extended。

---

## 二、环境准备

### 2.1 安装 Hugo Extended

去 [Hugo Releases](https://github.com/gohugoio/hugo/releases) 下载对应平台的 `hugo_extended_0.163.3_xxx.tar.gz`。

**Linux 示例：**

```bash
wget https://github.com/gohugoio/hugo/releases/download/v0.163.3/hugo_extended_0.163.3_linux-amd64.tar.gz
tar -xzf hugo_extended_0.163.3_linux-amd64.tar.gz
mv hugo ~/.local/bin/
```

验证一下：

```bash
hugo version
# 输出应为：hugo v0.163.3+extended+contracts ...
```

> ⚠️ 别用 apt 自带的 hugo，版本通常太旧。Blowfish 对 Hugo 版本有上下限要求，见 `themes/blowfish/config.toml` 里的 `[module.hugoVersion]`。

### 2.2 Git 与 GitHub 仓库

你需要：
- 一个 GitHub 账号
- Git 已配置好 SSH 或 HTTPS

新建一个名为 `<你的用户名>.github.io` 的仓库。这个命名约定是 GitHub Pages 的特殊规则——User Page 必须这么命名，之后就能通过 `https://<用户名>.github.io` 访问。

```bash
git clone git@github.com:<用户名>/<用户名>.github.io.git
cd <用户名>.github.io
```

---

## 三、初始化 Hugo 站点

### 3.1 创建站点骨架

```bash
hugo new site . --force
```

`--force` 是因为目录里已经有 `.git` 了，需要强制在当前目录初始化。

### 3.2 目录结构一瞥

完成后你会看到这些：

```
.
├── archetypes/        # hugo new 的模板
├── config/            # 站点配置（Hugo 0.158+ 新布局）
├── content/           # 文章放这里
├── data/              # 数据文件（可选）
├── layouts/           # 主题覆盖（可选）
├── static/            # 静态文件（favicon 等）
└── themes/            # 主题目录
```

我们的仓库实际结构差不多，只是配置统一放到了 `config/_default/` 下面。

---

## 四、安装 Blowfish 主题

推荐用 **Git submodule** 的方式，方便后续升级和版本锁定。

```bash
git submodule add git@github.com:nunocoracao/blowfish.git themes/blowfish
git submodule update --init --recursive
```

> 为什么不用 Hugo Modules？因为 submodule 更直观，GitHub Actions 里也多一行 `submodules: recursive` 就能拉取，出问题好排查。

锁定到特定版本（推荐）：

```bash
cd themes/blowfish
git fetch --tags
git checkout v2.104.0   # 换成你想用的 tag
cd ../..
```

记一下当前的版本号，后面 CI 配置要用。

### 4.1 主题需要的最低配置

Blowfish 的文档说需要一份最小配置才能跑起来。我们仓库里的 `config/_default/` 已经有完整的配置，核心文件：

| 文件 | 作用 |
|------|------|
| `hugo.toml` | 站点基础配置（baseURL、标题、分页、Markdown 等） |
| `languages.zh-cn.toml` | 语言/作者信息/社交链接 |
| `menus.zh-cn.toml` | 顶部导航和页脚菜单 |
| `params.toml` | 主题参数（首页布局、配色、功能开关） |
| `markup.toml` | Goldmark 渲染器 + KaTeX 分隔符 |

> 不想自己从零敲的话，可以直接复制本仓库的 `config/_default/` 目录。配置文件里该注释的地方都有注释，改起来不难。

---

## 五、配置站点

### 5.1 基础配置 `hugo.toml`

```toml
baseURL = 'https://<你的用户名>.github.io'
languageCode = 'zh-cn'
title = '你的站点标题'

[module]
  [[module.imports]]
    path = "github.com/nunocoracao/blowfish/v2"
```

如果你用的是 submodule 而不是 Hugo Modules，`module.imports` 那段可以不要，主题会自动从 `themes/blowfish` 读取。

### 5.2 语言与作者 `languages.zh-cn.toml`

```toml
[zh-cn]
  title = "你的站点标题"
  languageName = "中文"
  weight = 1

[zh-cn.params.author]
  name = "你的名字"
  image = "img/avatar.png"    # 放到 static/img/ 下面
  headline = "一句签名"
  bio = "关于你的一小段话"

# 社交链接，键名对应 assets/icons/ 里的 svg 或主题内置图标
[[zh-cn.params.author.links]]
  email = "mailto:你的邮箱"
  github = "https://github.com/你的用户名"
```

> Hugo 0.158 之后语言配置必须用 `locale` / `label` 字段，别再用已弃用的 `languageCode` / `languageName`，否则会报警告。

### 5.3 主题参数 `params.toml`

Blowfish 的外观几乎全靠这个文件控制。常用的几块：

```toml
[homepage]
  layout = "profile"        # profile / page / hero / card / background

[colorScheme]
  toggle = true             # 是否显示明暗切换按钮

[article]
  showDate = true
  showReadingTime = true
  showAuthor = true
  showBreadcrumbs = true

[footer]
  showCopyright = true
```

更多参数看 [Blowfish 配置文档](https://blowfish.page/docs/configuration/)。

### 5.4 菜单 `menus.zh-cn.toml`

```toml
[[main]]
  name = '文章'
  pageRef = '/posts'
  weight = 10

[[main]]
  name = '标签'
  pageRef = '/tags'
  weight = 20

# 带下拉的子菜单示例
[[main]]
  name = '工具链'
  pageRef = 'tags/工具链'
  parent = '文章'
  weight = 10
```

`pageRef` 的值要和文章 front matter 里的 `tags` 对应，不然会链到空页面。

---

## 六、写第一篇文章

### 6.1 用 archetype 新建

```bash
hugo new posts/hello-world.md
```

这会在 `content/posts/` 下生成一个带 front matter 的文件，模板来自 `archetypes/default.md`。

### 6.2 Front matter 模板

```yaml
---
title: "你好，世界"
date: 2026-07-29T15:30:00+08:00
draft: false
tags: ["学习笔记"]
description: "一句话摘要，用于列表和 SEO"
---
```

几点约定：

- `date` 用带时区的 ISO 8601 格式，时区 `+08:00`
- `draft: true` 的文章不会出现在生产构建（CI 不会加 `buildDrafts`）
- `tags` 写中文英文都行，但别太随意，否则标签页会爆炸

正文直接 Markdown 往下写就行。数学公式可以用 `$$...$$`（站点已开 KaTeX passthrough）。

---

## 七、本地预览

```bash
hugo server -D
```

`-D` 表示包含草稿。打开 `http://localhost:1313` 就能看效果，改文件会自动刷新。

生产风格构建（更接近 CI 产物）：

```bash
hugo --gc --minify
```

产物在 `public/` 目录，这个目录已经写在 `.gitignore` 里了，不要提交。

---

## 八、自动部署到 GitHub Pages

这是最爽的部分——**推代码即部署**，不需要本地构建，也不需要把 `public/` 提交上去。

### 8.1 Workflow 文件

在 `.github/workflows/hugo.yaml` 创建：

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.163.3
      TZ: Asia/Shanghai
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive    # 拉取 Blowfish 子模块
          fetch-depth: 0           # 让 GitInfo 正常工作

      - name: Setup Hugo
        uses: peace/actions-hugo@v3
        with:
          hugo-version: ${{ env.HUGO_VERSION }}
          extended: true            # 必须 extended

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Build
        env:
          HUGO_ENVIRONMENT: production
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 8.2 GitHub 仓库设置

进入仓库的 **Settings → Pages → Build and deployment**：
- Source 选 **GitHub Actions**

如果是 User Page（`<用户名>.github.io`），构建完会自动部署到根路径，不需要额外设置。

### 8.3 触发部署

```bash
git add .
git commit -m "init: hugo + blowfish + github pages"
git push origin main
```

推送后在仓库的 **Actions** 标签页能看到构建进度。大概 1-2 分钟就部署完了。

---

## 九、踩坑记录（都是真实踩过的）

### 9.1 Hugo 版本不匹配

**症状：** 构建报错 `module "blowfish" requires Hugo >= 0.158.0, <= 0.163.3`

**原因：** Blowfish 的 `config.toml` 里声明了兼容的 Hugo 版本区间，你的版本不在里面。

**解决：** 升级或降级 Hugo，或者升级 Blowfish。别硬改主题的 `config.toml` 来绕过——之后遇到兼容问题更头疼。

### 9.2 主题没拉下来（本地或 CI）

**症状：** `themes/blowfish/` 是空的，或者 CI 报错找不到主题。

**解决：**

```bash
git submodule update --init --recursive
```

CI 那边确认 `actions/checkout` 有 `submodules: recursive`。

### 9.3 CSS 样式丢失 / 页面裸奔

**症状：** 页面能访问但没有样式，浏览器控制台报 404 加载 CSS。

**原因：** `baseURL` 没配置对。GitHub Pages 如果挂在子路径（Project Page 而不是 User Page），`baseURL` 必须带那个子路径。

**解决：**
- User Page（`<用户名>.github.io`）：`baseURL = "https://<用户名>.github.io/"`
- Project Page（`<用户名>.github.io/项目名`）：`baseURL = "https://<用户名>.github.io/项目名/"`

末尾的 `/` 别漏。

### 9.4 数学公式没渲染

站点 `markup.toml` 里已经配了 passthrough，用 `$$...$$` 或 `\[...\]` 即可。但如果你用的是行内公式 `\(...\)`，确认配置里 `inline` 的分隔符也对上了。

### 9.5 Git submodule 指针没提交

**症状：** 你本地主题更新了，但 CI 拉下来的还是旧版本。

**原因：** submodule 的 gitlink（那个 SHA 指针）没跟着提交。

**解决：**

```bash
git add themes/blowfish
git commit -m "chore: bump blowfish to v2.104.0"
```

子模块的指针变更必须进同一次提交，否则 CI 会 checkout 到错误版本。

---

## 十、进阶配置（可选）

### 10.1 自定义域名

1. 在 `static/` 下加一个 `CNAME` 文件，内容是你的域名
2. DNS 里配 A 记录指向 GitHub Pages 的 IP
3. 仓库 Settings → Pages → Custom domain 填上域名

### 10.2 评论系统

Blowfish 支持 Giscus、Disqus、Utterances 等。以 Giscus 为例，在 `params.toml` 里：

```toml
[comments]
  provider = "giscus"
  [comments.giscus]
    repo = "用户名/仓库名"
    repoId = "..."
    category = "Announcements"
    categoryId = "..."
```

去 [giscus.app](https://giscus.app) 跟着引导走就行。

### 10.3  analytics

Google Analytics、Plausible、Umami 都支持，在 `params.toml` 对应位置填上 ID 或 src 即可。

### 10.4 升级 Blowfish

```bash
cd themes/blowfish
git fetch --tags
git checkout v2.x.x    # 目标版本
cd ../..

# 然后读一下新版本的 config.toml 的 [module.hugoVersion]
# 如果 Hugo 版本要求变了，同步改 CI 的 HUGO_VERSION

git add themes/blowfish
git commit -m "chore: bump blowfish to v2.x.x"
```

---

## 十一、常用命令速查

| 任务 | 命令 |
|------|------|
| 新建文章 | `hugo new posts/文件名.md` |
| 本地预览（含草稿） | `hugo server -D` |
| 仅本机访问 | `hugo server -D --bind 127.0.0.1` |
| 生产构建 | `hugo --gc --minify` |
| 清理构建产物 | `rm -rf public resources .hugo_cache` |

---

## 十二、参考链接

- 本仓库源码：<https://github.com/XeriChen/XeriChen.github.io>
- Hugo 文档：<https://gohugo.io/documentation/>
- Blowfish 文档：<https://blowfish.page/docs/configuration/>
- Blowfish Releases：<https://github.com/nunocoracao/blowfish/releases>
- GitHub Pages 文档：<https://docs.github.com/en/pages>

---

以上就是完整的搭建流程。如果你是从零开始，照着做大概 30 分钟内能跑起来一个本地预览，推一次代码就能上线。主题这块 Blowfish 可玩性很高，后面可以慢慢折腾配色、布局和 shortcodes。

有任何问题可以在仓库开 Issue，或者直接去看 Blowfish 的 GitHub Discussions——作者回复挺积极的。
