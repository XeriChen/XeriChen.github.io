# XeriChen 的个人博客

基于 [Hugo](https://gohugo.io) Extended + [Blowfish](https://blowfish.page/) 主题，部署在 GitHub Pages。

**站点：** [https://xerichen.github.io](https://xerichen.github.io)

## Agent 说明

给 AI / 自动化助手维护本仓库用的规则与约定见 **[AGENTS.md](./AGENTS.md)**（工具链版本、文章 front matter、菜单标签、主题升级、CI、`public/` 忽略规则等）。

## 工具链（摘要）

| 组件 | 版本约定 |
|------|----------|
| Hugo（CI / 推荐本地） | **0.163.3 Extended** |
| Blowfish | **v2.104.0**（git submodule） |
| 主题支持的 Hugo | min `0.158.0`，max `0.163.3` |

本地安装示例（Linux amd64，与 CI 对齐）：

```bash
cd /tmp
curl -fsSL -o hugo.tar.gz \
  https://github.com/gohugoio/hugo/releases/download/v0.163.3/hugo_extended_0.163.3_linux-amd64.tar.gz
tar -xzf hugo.tar.gz hugo
mkdir -p ~/.local/bin && install -m 755 hugo ~/.local/bin/hugo
hugo version   # 应含 v0.163.3+extended
```

## 克隆

```bash
git clone --recurse-submodules git@github.com:XeriChen/XeriChen.github.io.git
# 若 themes/blowfish 为空：
git submodule update --init --recursive
```

## 写文章

```bash
hugo new posts/your-post-slug.md
# 编辑 content/posts/… 的 front matter：title / date / draft / tags / description
```

顶栏「文章」下的标签快捷入口在 `config/_default/menus.zh-cn.toml`（当前含：工具链、MBTI、LOVE、旅行）。

## 本地构建与预览

```bash
hugo --gc --minify    # 生产风格构建 → 输出到 public/（已 gitignore）
hugo server -D        # 本地预览（含草稿）
```

**说明：** `public/` 仅为本地/CI 构建产物，已由 `.gitignore` 忽略，**不要提交**。线上站点由 `.github/workflows/hugo.yaml` 在 push `main` 时构建并部署到 GitHub Pages。
