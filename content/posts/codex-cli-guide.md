---
title: "Codex CLI 完整使用指南：OpenAI 的终端编程助手"
date: 2026-07-19T16:42:36+08:00
draft: false
tags: ["学习笔记", "工具链", "AI", "Codex"]
description: "从安装、ChatGPT/API 认证、TUI 交互到沙箱审批、Skills、MCP、codex exec 与 code review——基于官方文档与 codex-cli 0.144.5 的中文全功能指南。"
---

> **说明：** 本文基于 OpenAI Codex 官方文档（[developers.openai.com/codex](https://developers.openai.com/codex)、GitHub [openai/codex](https://github.com/openai/codex)）与本机 CLI 帮助整理。撰写时本地版本为 **`codex-cli 0.144.5`**。不同版本命令与 feature 成熟度可能变化，以 `codex --help` 与官方文档为准。  
> 安装：`curl -fsSL https://chatgpt.com/codex/install.sh | sh`  
> 更新：`codex update`

## 一、Codex CLI 是什么

**Codex CLI** 是 OpenAI 的**本地终端编码 agent**。它在你的机器上运行，能读仓库、改文件、执行 shell、做代码评审，并把可重复流程打包成 Skills / Plugins。

不要和同名生态里的其它形态搞混：

| 产品形态 | 入口 / 说明 |
|----------|-------------|
| **Codex CLI** | `codex` —— 本文主角，终端 TUI + 脚本 |
| **IDE 扩展** | VS Code / Cursor / Windsurf 等，与 CLI 共享配置层 |
| **Desktop App** | `codex app` 或 ChatGPT 桌面端里的 Codex |
| **Codex Cloud / Web** | 云端任务；可用 `codex cloud` / `codex apply` 与本地衔接 |

三种最常用用法：

| 用法 | 命令 | 适合 |
|------|------|------|
| 交互式 TUI | `codex` / `codex "任务"` | 日常开发、调试、迭代 |
| 非交互 / 脚本 | `codex exec "..."` | CI、管道、批量任务 |
| 代码评审 | `codex review` / TUI 内 `/review` | 提交前、PR 前只读审查 |

和「在网页里贴代码」不同：Codex 直接在你的工作目录动手，diff 与命令会进入**会话（session）**，可用 `codex resume` 续聊。

---

## 二、安装与更新

### 2.1 系统要求（官方）

| 要求 | 说明 |
|------|------|
| 操作系统 | macOS 12+；Ubuntu 20.04+ / Debian 10+；Windows 11 推荐 **WSL2** |
| Git | 可选但推荐（2.23+，便于 PR 相关能力） |
| 内存 | 最低约 4GB，推荐 8GB+ |

### 2.2 安装方式

**macOS / Linux（独立安装器，推荐）：**

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
```

**Windows PowerShell：**

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"
```

**包管理器：**

```bash
# npm 全局
npm install -g @openai/codex

# Homebrew（cask）
brew install --cask codex
```

也可从 [GitHub Releases](https://github.com/openai/codex/releases/latest) 下载对应平台二进制（musl Linux、Apple Silicon 等）。

验证：

```bash
codex --version
# 例：codex-cli 0.144.5
```

### 2.3 更新与体检

```bash
codex update          # 升到最新
codex doctor          # 安装、配置、认证、运行时健康检查
codex doctor --json   # 机器可读报告（已脱敏）
codex doctor --summary
```

---

## 三、认证

Codex 支持两种主要登录方式（本地 CLI / IDE / 桌面端）：

1. **Sign in with ChatGPT** —— 走 Plus / Pro / Business / Edu / Enterprise 等订阅额度（推荐日常）  
2. **API Key** —— 按 OpenAI Platform 用量计费，适合自动化  

Cloud 侧通常要求 ChatGPT 登录。

### 3.1 交互登录

```bash
codex                  # 首次会引导登录
codex login            # 重新登录
codex login --device-auth   # 设备码（无本机浏览器时）
codex login status
codex logout
```

API Key 从 stdin 写入（避免出现在 shell 历史参数里）：

```bash
printenv OPENAI_API_KEY | codex login --with-api-key
```

Enterprise 也可使用 access token 做非交互本地工作流：

```bash
printenv CODEX_ACCESS_TOKEN | codex login --with-access-token
```

### 3.2 凭证存放

- 默认缓存：`~/.codex/auth.json` 或系统钥匙串  
- 配置项 `cli_auth_credentials_store`：`file` | `keyring` | `auto`  

**把 `auth.json` 当密码对待**：不要提交、不要贴进工单或聊天。

### 3.3 自动化里的密钥

`codex exec` 可对单次调用使用：

```bash
CODEX_API_KEY=<api-key> codex exec --json "triage open bug reports"
```

注意：

- `CODEX_API_KEY` **仅**在 `codex exec` 路径有文档支持。  
- 不要在会跑不可信代码的同一 job 环境里把 API Key 设成全局 env（构建脚本可读走密钥）。  
- GitHub 上优先考虑官方 [openai/codex-action](https://github.com/openai/codex-action)，由 Action 侧代理密钥。

### 3.4 自定义模型提供方（概念）

在 `config.toml` 的 `[model_providers.*]` 中可接代理或本地模型，常见认证形态：

- `requires_openai_auth = true` —— 走 OpenAI/ChatGPT 登录（经代理）  
- `env_key = "MY_PROVIDER_KEY"` —— 读环境变量  
- 两者皆无 —— 视为无需认证（如本地 Ollama）

本地开源提供方也可：

```bash
codex --oss                      # 使用配置的 oss_provider 或交互选择
codex --oss --local-provider ollama
```

---

## 四、第一次上手

### 4.1 启动交互会话

```bash
cd your-repo
codex

# 带初始任务
codex "解释这个仓库的架构，并列出三个最大风险点"

# 指定工作目录
codex -C ~/projects/my-app "修掉 failing test"

# 指定模型
codex -m gpt-5.6 "重构错误处理"

# 附带图片（报错截图、设计稿）
codex -i ./error.png "根据截图修 UI"

# 启用 live 网页搜索（默认多为 cached）
codex --search "对照最新 React 文档改这里的 API"
```

### 4.2 TUI 里先会的几件事

官方欢迎屏常提示：

| 命令 | 作用 |
|------|------|
| `/init` | 生成项目 `AGENTS.md` 脚手架 |
| `/status` | 查看当前会话配置与用量相关信息 |
| `/approvals`（或权限相关入口） | 选择 Codex 无需询问即可做的事 |
| `/model` | 切换模型与 reasoning effort |
| `/review` | 进入代码评审模式 |

用 `@filename` 把文件钉进提示（例如：`Improve documentation in @README.md`）。  
底部常有 context 余量与 `?` 快捷键提示。

### 4.3 控制「能干什么」

启动时或配置里设定沙箱与审批（详见第十二节）：

```bash
codex -s workspace-write -a on-request
codex -s read-only                      # 更保守，适合先摸清代码
# 危险：跳过审批与沙箱（仅限外部已隔离环境）
codex --dangerously-bypass-approvals-and-sandbox
# 别名常见为 --yolo（官方文档与 developer-commands 中有记载）
```

---

## 五、核心能力一览

| 能力 | 说明 |
|------|------|
| 读改代码 / 跑命令 | 本地仓库闭环 |
| 沙箱 + 审批策略 | 限制写盘与网络，控制是否弹窗 |
| Session | 持久会话；resume / fork / archive |
| AGENTS.md | 项目级长期指令 |
| Skills | 可复用工作流（agentskills 标准） |
| MCP | 外接工具与数据源 |
| Plugins | 打包 skills / 连接器分发 |
| Code review | `codex review` 或 `/review`，默认不改树 |
| 非交互 exec | CI、管道、JSONL、JSON Schema 输出 |
| Web search | cached / live 等模式 |
| Multi-agent | 子代理协作（feature `multi_agent`） |
| Cloud 衔接 | 浏览云任务、`apply` 落本地 |
| Doctor / features | 诊断与功能开关 |

---

## 六、全局 CLI 标志（实用子集）

无子命令时，选项进入交互 CLI。多数标志也会传到 `exec` 等命令。

| 标志 | 含义 |
|------|------|
| `-m, --model` | 覆盖默认模型 |
| `-C, --cd` | 工作根目录 |
| `--add-dir` | 额外可写目录（可重复） |
| `-s, --sandbox` | `read-only` \| `workspace-write` \| `danger-full-access` |
| `-a, --ask-for-approval` | `untrusted` \| `on-request` \| `never` |
| `-c, --config key=value` | 覆盖 `config.toml`（值按 TOML 解析） |
| `-p, --profile` | 叠一层 `$CODEX_HOME/<name>.config.toml` |
| `-i, --image` | 初始提示附带图片 |
| `--search` | live 网页搜索 |
| `--oss` / `--local-provider` | 本地开源模型提供方 |
| `--enable` / `--disable` | 开关 feature flag |
| `--strict-config` | 未知 config 字段直接报错 |
| `--no-alt-screen` | 不用 alternate screen，保留终端滚动历史 |
| `--remote` | 连远程 app-server（ws/wss/unix） |
| `--dangerously-bypass-approvals-and-sandbox` | 跳过审批与沙箱（极危） |
| `--dangerously-bypass-hook-trust` | 跳过 hook 信任（仅可信自动化） |

配置覆盖示例：

```bash
codex -c model="gpt-5.6" -c model_reasoning_effort=\"high\"
codex -c 'sandbox_mode="read-only"'
```

完整标志表见官方 [Developer commands](https://developers.openai.com/codex/developer-commands)。

---

## 七、子命令地图

```text
codex                 # 交互 TUI
codex exec | e        # 非交互
codex review          # 非交互代码评审
codex login / logout
codex resume / fork / archive / unarchive / delete
codex mcp              # 管理 MCP 服务器
codex plugin           # 插件与 marketplace
codex mcp-server       # 把 Codex 自身作为 MCP server（stdio）
codex apply | a        # 把云端任务 diff 应用到本地
codex cloud            # [Experimental] 浏览云任务
codex sandbox          # 在 Codex 沙箱中跑指定命令
codex doctor
codex update
codex features list|enable|disable
codex completion bash|zsh|fish|...
codex app / app-server / remote-control / exec-server  # 桌面或实验服务
```

查看子命令帮助：

```bash
codex exec --help
codex review --help
codex mcp --help
codex features list
```

---

## 八、配置体系

### 8.1 文件位置

| 路径 | 作用 |
|------|------|
| `~/.codex/config.toml` | 用户默认配置 |
| `$CODEX_HOME/<profile>.config.toml` | `--profile` 叠加层 |
| 项目内 `.codex/config.toml` | 项目覆盖（**仅 trusted 项目加载**） |
| `/etc/codex/config.toml` | 系统级（Unix，若存在） |
| 托管 `requirements.toml` | 企业强制约束（如禁止 `never` / full-access） |

`CODEX_HOME` 默认 `~/.codex`。

### 8.2 优先级（高 → 低）

1. CLI 标志与 `-c`  
2. 可信项目的 `.codex/config.toml`（从仓库根到 cwd，**越近越优先**）  
3. `--profile` 配置文件  
4. `~/.codex/config.toml`  
5. 系统 config  
6. 内置默认  

未信任项目会跳过项目级 `.codex/`（含 hooks、rules）；用户级配置仍加载。

### 8.3 常用配置示例

```toml
# ~/.codex/config.toml

model = "gpt-5.6"
model_reasoning_effort = "high"   # 模型支持时
approval_policy = "on-request"    # untrusted | on-request | never
sandbox_mode = "workspace-write"  # read-only | workspace-write | danger-full-access

# 网页搜索：cached（默认）| indexed | live | disabled
web_search = "cached"

personality = "pragmatic"         # friendly | pragmatic | none 等

# 可选 feature
[features]
multi_agent = true
hooks = true
memories = false                  # experimental
shell_tool = true

# 限制传给子进程的环境变量
[shell_environment_policy]
include_only = ["PATH", "HOME"]

# 日志目录（显式设置可启用明文 TUI 日志）
# log_dir = "/absolute/path/to/codex-logs"
```

启用 feature 的其它方式：

```bash
codex --enable multi_agent
codex features enable memories
codex features list
```

### 8.4 Profile

```bash
# 使用 ~/.codex/ci.config.toml 叠在用户配置之上
codex -p ci "run the release checklist"
```

适合「日常宽松 / CI 严格」分档。

---

## 九、AGENTS.md：教 Codex 怎么在你仓库干活

`AGENTS.md` 是给 agent 的项目说明书：构建命令、测试方式、目录边界、评审标准等。

```bash
# TUI 内
/init
```

会在当前目录脚手架一份 `AGENTS.md`，**务必按团队真实流程改写**。也可手写放在仓库根（或子目录，按 Codex 发现规则加载）。

与本博客仓库的 `AGENTS.md` 是同一类文件：给 Grok / Claude / Codex 等 agent 读的约定，不是给终端用户的 README。

团队还可在 `AGENTS.md` 里引用 `code_review.md` 等，让 `/review` 行为一致。

---

## 十、Skills：可复用工作流

Skills 基于 [Agent Skills](https://agentskills.io) 思路：把「何时用 + 怎么做」写进 `SKILL.md`，需要时再加载全文（progressive disclosure，省 context）。

### 10.1 目录结构

```text
my-skill/
  SKILL.md          # 必需：YAML frontmatter + 指令
  scripts/          # 可选
  references/       # 可选
  assets/           # 可选
  agents/openai.yaml  # 可选：UI、隐式调用策略、MCP 依赖
```

最小 `SKILL.md`：

```markdown
---
name: skill-name
description: 说明何时该用、何时不该用。写清触发词。
---

1. 第一步……
2. 第二步……
```

### 10.2 存放位置（常见）

| 范围 | 路径 |
|------|------|
| 仓库 / 当前目录 | `$CWD/.agents/skills`、向上直到 `$REPO_ROOT/.agents/skills` |
| 用户 | `~/.agents/skills` |
| 管理员 | `/etc/codex/skills` |
| 系统内置 | 随 Codex 分发（如 skill-creator） |

同名 skill 不会合并，选择器里可同时出现。

### 10.3 调用方式

1. **显式**：TUI 中 `/skills`，或 `$skill-name` / `$skill-creator`  
2. **隐式**：任务描述匹配 `description` 时自动选用  

禁止隐式可用 metadata `allow_implicit_invocation: false`。

创建与安装：

```text
$skill-creator          # 交互创建 skill
$skill-installer linear # 安装 curated skill 示例
```

也可手动建目录；改完一般会自动发现，不行就重启 Codex。

禁用某 skill（不删文件）：

```toml
[[skills.config]]
path = "/path/to/skill/SKILL.md"
enabled = false
```

### 10.4 与 Plugins

Skill 适合本地/仓库内工作流；要跨团队分发、捆绑 MCP/连接器，用 **plugin**（`codex plugin` / marketplace）。

---

## 十一、MCP 与 Plugins

### 11.1 MCP

```bash
codex mcp list
codex mcp get <name>

# stdio 服务器（-- 之后是启动命令）
codex mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem /path

# 带环境变量
codex mcp add myserver --env API_KEY=xxx -- /path/to/server

# Streamable HTTP
codex mcp add remote --url https://mcp.example.com/mcp
codex mcp add remote --url https://mcp.example.com/mcp --bearer-token-env-var MY_TOKEN

codex mcp login <name>    # 需要 OAuth 时
codex mcp remove <name>
```

TUI 内也可用 `/mcp` 查看当前会话可用工具。  
若某 MCP 配置为 `required = true` 且启动失败，`codex exec` 会直接失败而不是静默继续。

### 11.2 Plugins

```bash
codex plugin list
codex plugin marketplace   # add / list / upgrade / remove 源
codex plugin add <...>     # 从已配置 marketplace 安装
codex plugin remove <...>
```

Plugins 可打包 skills、连接器与展示资源，并与 ChatGPT Work mode / 桌面端生态衔接。

---

## 十二、沙箱与审批（务必读）

Codex 把「模型想跑的命令」关在策略里执行，而不是默认裸奔。

### 12.1 沙箱模式（`-s` / `sandbox_mode`）

| 模式 | 含义（概念） |
|------|----------------|
| `read-only` | 以只读为主，适合探索、评审、解释 |
| `workspace-write` | 可写工作区（及策略允许的路径），日常开发常用 |
| `danger-full-access` | 接近关闭沙箱限制，风险最高 |

可写根上的 **`.git` / `.codex` 等路径**可能有额外保护；网络默认也因模式而异。细节见官方 [Agent approvals & security](https://developers.openai.com/codex/agent-approvals-security)。

额外可写目录：

```bash
codex --add-dir ../shared-libs --add-dir /tmp/codex-scratch
```

### 12.2 审批策略（`-a` / `approval_policy`）

| 策略 | 行为 |
|------|------|
| `untrusted` | 仅「受信」类命令免询问，其它升级问人 |
| `on-request` | 由模型决定何时请求批准（常用默认思路） |
| `never` | 不询问；失败直接回给模型（适合严格受控自动化） |

### 12.3 危险开关

```bash
# 跳过审批 + 沙箱 —— 仅当你已有外部容器/VM 隔离时
codex --dangerously-bypass-approvals-and-sandbox
```

`codex exec` 文档明确：默认偏 **read-only**；自动化应显式给最小权限：

```bash
codex exec --sandbox workspace-write "apply the minimal fix and run tests"
```

旧的 `--full-auto` 属于兼容弃用路径，新脚本请写清楚 `--sandbox`。

### 12.4 Execpolicy rules

用户/项目可放 execpolicy `.rules`，用规则允许/拒绝命令前缀。  
单次忽略：`codex exec --ignore-rules`（只在你完全可控的环境使用）。

本机也可用：

```bash
codex sandbox -- <command...>   # 在 Codex 提供的沙箱里跑命令
```

---

## 十三、非交互模式：`codex exec`

### 13.1 基本行为

```bash
codex exec "summarize the repository structure and list the top 5 risky areas"
```

- **进度** → `stderr`  
- **最终回复** → `stdout`（方便管道）

```bash
codex exec "generate release notes for the last 10 commits" | tee release-notes.md
```

### 13.2 常用选项

```bash
# 不落盘会话
codex exec --ephemeral "triage this repository"

# JSONL 事件流（stdout）
codex exec --json "summarize the repo structure" | jq

# 最终消息另存
codex exec "..." -o ./last-message.txt

# 结构化输出（JSON Schema 文件）
codex exec "Extract project metadata" \
  --output-schema ./schema.json \
  -o ./project-metadata.json

# 沙箱与工作目录
codex exec -C ~/proj -s workspace-write "fix failing tests"

# 非 git 仓库（需你确认安全）
codex exec --skip-git-repo-check "..."

# 不加载用户 config / 不加载 rules
codex exec --ignore-user-config --ignore-rules "..."
```

默认要求在 **Git 仓库**内运行，降低误伤范围。

### 13.3 管道

```bash
# 提示词 + stdin 作为附加上下文
curl -s https://example.com/data.json \
  | codex exec "format the top 20 items into a markdown table" \
  > table.md

# stdin 整段作为 prompt
some-command | codex exec -
```

### 13.4 续跑

```bash
codex exec "review the change for race conditions"
codex exec resume --last "fix the race conditions you found"
# 或
codex exec resume <SESSION_ID> "continue"
```

### 13.5 CI 要点

- 权限最小化：`read-only` 做分析，确认后才 `workspace-write`。  
- 密钥单次注入，避免和不可信 `npm install` 同环境暴露。  
- 可用官方 **Codex GitHub Action** 做安装 + API 代理 + 跑 prompt。  
- 常见模式：失败 CI → 只读 checkout → Codex 出 patch artifact → 另一 job 开 PR（写权限与 API Key 分离）。

---

## 十四、代码评审：`codex review` 与 `/review`

评审模式以**找问题**为主：严重度、文件/行引用、风险与测试缺口；默认**不修改工作树**。

### 14.1 CLI

```bash
# 未提交变更（staged + unstaged + untracked）
codex review --uncommitted

# 相对 base 分支（PR 视角）
codex review --base main

# 某个 commit
codex review --commit abcdef1 --title "Fix auth timeout"

# 自定义评审说明
codex review "Focus on security and race conditions"
```

### 14.2 TUI

```text
/review
```

可选：相对 base、未提交变更、某次 commit、自定义指令。  
团队标准可写在 `code_review.md` 并由 `AGENTS.md` 引用。

---

## 十五、会话生命周期

交互会话会记录在 `CODEX_HOME` 下（sessions 等），便于续作。

```bash
# 选择器恢复（默认按 cwd 过滤）
codex resume
codex resume --last
codex resume <SESSION_ID_OR_NAME> "继续上次的迁移"
codex resume --all                      # 显示全部会话
codex resume --include-non-interactive  # 包含 exec 会话

# 从历史分叉新会话
codex fork
codex fork --last

# 归档 / 恢复归档 / 永久删除
codex archive <id-or-name>
codex unarchive <id-or-name>
codex delete <id-or-name>
```

无头侧对应 `codex exec resume`。

---

## 十六、Cloud 与本地衔接（简述）

| 命令 | 说明 |
|------|------|
| `codex cloud` | **Experimental**：在终端浏览/操作 Cloud 任务 |
| `codex apply <TASK_ID>` | 将 Codex agent 产出的最新 diff `git apply` 到本地 |

适合「云端跑长任务 → 本地审 diff」。Cloud 账号安全建议开启 MFA（官方安全文档有专门说明）。

---

## 十七、推荐工作流

### 17.1 日常修 bug

```bash
cd your-repo
codex -s workspace-write -a on-request
```

提示词示例：

```text
复现并修复 tests/auth 相关失败。改动尽量小，跑相关测试，最后用条目说明原因。
不要重构无关文件。
```

### 17.2 先摸清再动手

```bash
codex -s read-only "画出请求从 HTTP 到数据库的路径，标出 3 个最可能的故障点"
```

### 17.3 提交前评审

```bash
codex review --uncommitted
# 或相对 main
codex review --base main
```

### 17.4 团队标准化

1. 根目录 `AGENTS.md`（`/init` 后改）  
2. `.agents/skills/` 放发布、评审、迁移流程  
3. 可信项目下 `.codex/config.toml` 统一沙箱默认值  
4. CI 用 `codex exec` + 最小 sandbox + schema 输出  

### 17.5 管道摘要

```bash
git log -20 --oneline | codex exec "写成面向用户的更新说明（中文）" > notes.md
```

### 17.6 与 Grok Build 对照（便于迁移心智）

| 维度 | Codex CLI | Grok Build（参考） |
|------|-----------|---------------------|
| 厂商 | OpenAI | xAI |
| 交互入口 | `codex` | `grok` |
| 无头 | `codex exec` | `grok -p` / `--single` |
| 配置 | `~/.codex/config.toml` | `~/.grok/config.toml` |
| 项目指令 | `AGENTS.md`、`.agents/skills` | `AGENTS.md`、`.grok/skills` |
| 沙箱命名 | `read-only` / `workspace-write` / `danger-full-access` | `workspace` / `read-only` / `strict` 等 |
| 一键放开 | `--dangerously-bypass-approvals-and-sandbox` | `--yolo` / always-approve |
| 代码评审 | 一等公民 `codex review` | 多靠 prompt / skill |
| 登录 | ChatGPT 或 API Key | grok.com / `XAI_API_KEY` |

二者都是「终端里的 agent」；命令与目录约定不同，但 **AGENTS.md + 最小权限 + 可复现脚本** 的实践是通用的。

---

## 十八、Shell 补全与终端体验

```bash
codex completion zsh > ~/.zsh/completions/_codex
codex completion bash
codex completion fish
```

长提示可交给 `VISUAL` / `EDITOR` 外部编辑器（TUI 能力之一）。  
需要保留终端原生滚动时用 `--no-alt-screen`。

调试日志：

```bash
codex -c log_dir=./.codex-log
# 视配置产生 codex-tui.log 等
RUST_LOG=info codex exec "..."   # exec 默认偏 error，可调高
```

---

## 十九、路径与文件速查

| 路径 | 内容 |
|------|------|
| `~/.codex/config.toml` | 主配置 |
| `~/.codex/auth.json` | 登录凭证（敏感） |
| `~/.codex/sessions/`（及内部存储） | 会话数据 |
| `~/.codex/skills/` / 系统 skills | 技能相关缓存或内置 |
| `~/.codex/rules/` | execpolicy 等规则 |
| `~/.agents/skills/` | 用户 skills |
| `<repo>/.agents/skills/` | 项目 skills |
| `<repo>/.codex/config.toml` | 项目配置（需 trust） |
| `<repo>/AGENTS.md` | 项目 agent 说明 |

---

## 二十、常见问题

**Q：登录失败或凭证过期？**  
`codex logout` 后 `codex login`；检查系统时间与网络；企业环境看是否强制 workspace / SSO。

**Q：项目里的 `.codex` 不生效？**  
项目可能未标记 trusted；不信任时会跳过项目层 config/hooks/rules。

**Q：exec 不能写文件？**  
默认偏只读。显式：`codex exec -s workspace-write "..."`。

**Q：想完全自动、从不询问？**  
`approval_policy = "never"` 或 `-a never`，并清醒选择 sandbox。不要在个人笔记本上对不可信仓库开 `danger-full-access` + yolo。

**Q：Windows 原生还是 WSL2？**  
官方系统要求强调 Windows 11 **via WSL2**；原生 Windows 另有 sandbox elevated 等设置，见 config 文档中的 `[windows]`。

**Q：如何确认当前功能开了没？**  
`codex features list` 看 stage 与 effective state；`codex doctor` 看整体健康。

**Q：Skill 不出现？**  
检查路径与 `name`；看是否被 `[[skills.config]] enabled=false`；重启 CLI；技能过多时初始列表可能被截断（仍可显式 `$name`）。

---

## 二十一、参考链接

| 资源 | URL |
|------|-----|
| Codex 文档总入口 | https://developers.openai.com/codex |
| CLI 概览 | https://developers.openai.com/codex/cli |
| Developer commands | https://developers.openai.com/codex/developer-commands |
| 认证 | https://developers.openai.com/codex/auth |
| 配置基础 | https://developers.openai.com/codex/config-basic |
| 非交互模式 | https://developers.openai.com/codex/noninteractive |
| Skills | https://developers.openai.com/codex/skills |
| 审批与沙箱 | https://developers.openai.com/codex/agent-approvals-security |
| 最佳实践 | https://developers.openai.com/codex/learn/best-practices |
| GitHub 仓库 | https://github.com/openai/codex |
| Codex GitHub Action | https://github.com/openai/codex-action |
| Agent Skills 规范 | https://agentskills.io |

---

## 二十二、小结

Codex CLI 把「对话模型 + 本地工具 + 沙箱策略 + 可扩展 Skills/MCP」收成一条终端工作流：

1. **装好 → `codex login` → 在仓库里 `codex`**  
2. **用 `AGENTS.md` + Skills** 固化团队习惯  
3. **用 sandbox / approval** 控制风险，而不是默认全开  
4. **用 `review` 守提交门，用 `exec` 接 CI 与管道**  
5. **用 MCP / Plugins** 接到真实工单、文档与内部系统  

从官方欢迎语开始的最短路径：

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
cd your-repo
codex
# /init → 改 AGENTS.md → 描述你的第一个任务
```

---

*若本文与你本机 `codex --version` 或官方文档冲突，以官方与 CLI help 为准。*
