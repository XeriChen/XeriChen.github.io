---
title: "Grok Build 完整使用指南：终端里的 AI 编程助手"
date: 2026-07-19T13:57:43+08:00
draft: false
tags: ["学习笔记", "工具链", "AI", "Grok"]
description: "从安装、认证、TUI 交互到 Skills、MCP、子代理、Plan Mode、无头模式与沙箱——基于官方文档的 Grok Build 全功能中文指南。"
---

> **说明：** 本文基于 Grok Build 官方用户手册（`~/.grok/docs/user-guide/`）与 CLI 实际能力整理，结合日常在终端中的使用经验。文中命令与配置以文档为准；不同版本界面细节可能略有差异。撰写时本地 CLI 版本为 `grok 0.2.103`。  
> 在线文档入口：<https://docs.x.ai/build/overview>  
> 安装与更新：`curl -fsSL https://x.ai/cli/install.sh | bash` / `grok update`

## 一、Grok Build 是什么

**Grok Build** 是 xAI 推出的**终端原生 AI 编程助手**（TUI，Terminal User Interface）。它能理解你的代码仓库、读写文件、执行 shell、搜索网页、管理任务，并在需要时拉起子代理并行工作。

三种常见用法：

| 用法 | 入口 | 适合场景 |
|------|------|----------|
| 交互式 TUI | `grok` | 日常开发、改 bug、写功能 |
| 无头 / 脚本 | `grok -p "..."` | CI/CD、自动化、管道处理 |
| Agent 模式（ACP） | `grok agent stdio` 等 | IDE / 编辑器集成 |

和「只在聊天框里贴代码」不同，Grok Build 直接在你的项目目录里动手：diff 编辑、跑测试、提交前自检，对话与工具调用会完整进 **Session**，可随时 `/resume`。

---

## 二、安装与更新

### 2.1 安装

**macOS / Linux / Windows（Git Bash）：**

```bash
curl -fsSL https://x.ai/cli/install.sh | bash
```

指定版本：

```bash
curl -fsSL https://x.ai/cli/install.sh | bash -s 0.1.42
```

**Windows PowerShell：**

```powershell
irm https://x.ai/cli/install.ps1 | iex
```

安装后验证：

```bash
grok --version
```

### 2.2 更新

```bash
grok update
```

也可在 `~/.grok/config.toml` 中配置启动时自动检查更新：

```toml
[cli]
auto_update = true
```

---

## 三、认证

### 3.1 浏览器登录（默认）

首次运行 `grok` 会打开浏览器完成登录，凭证写入 `~/.grok/auth.json`，之后自动刷新。

```bash
grok login              # 重新登录（默认 OAuth）
grok login --device-auth  # 无浏览器环境：设备码流程
grok logout             # 退出并清除本地凭证
```

### 3.2 API Key（CI / 无界面）

从 [console.x.ai](https://console.x.ai) 申请密钥：

```bash
export XAI_API_KEY="xai-..."
grok
```

注意：若已有交互登录的 session token，**session 优先于 API Key**。要强制用 Key，先 `grok logout`。

### 3.3 企业 OIDC / 外部认证脚本

支持通过 IdP（Okta、Azure AD 等）OIDC，或自定义 `auth_provider_command` 脚本输出 token。适合公司内网、无浏览器沙箱。配置见 `~/.grok/config.toml` 的 `[grok_com_config.oidc]` / `[auth]`，细节可在 TUI 内 `/docs Authentication` 或阅读官方 `02-authentication` 文档。

**凭证优先级（从高到低）：** 模型级 `api_key`/`env_key` → 活动 session token → `XAI_API_KEY`。

---

## 四、第一次上手

### 4.1 启动

```bash
# 进入当前目录的交互会话
grok

# 带初始任务
grok "修一下登录失败的单测并跑通"

# 指定工作目录
grok --cwd ~/projects/my-app

# 使用 git worktree 隔离改动（注意 --worktree=name 的写法）
grok --worktree=feat "重构模块 X"

# 从 main 建 worktree 再开工
grok -w --ref main "实现某功能"

# 继续最近一次会话
grok -c

# 恢复指定会话
grok --resume <session-id>

# 全部自动批准工具（谨慎）
grok --yolo
# 或
grok --always-approve
```

### 4.2 界面结构

- **Scrollback（回滚区）**：对话历史、思考块、工具调用、diff、任务列表  
- **Prompt（输入区）**：底部输入框  

`Tab` / `Space` 在两区之间切换焦点。`Enter` 发送；`Shift+Enter` / `Alt+Enter` 换行（多行模式见 `/multiline`）。

### 4.3 用 `@` 引用文件

```text
@src/main.rs              # 附加整个文件
@src/main.rs:10-50        # 只附加 10–50 行
@src/                     # 浏览目录
@!.env                    # 搜索/附加被 gitignore 的隐藏文件（! 前缀）
```

输入 `@` 会弹出模糊文件选择器（默认尊重 `.gitignore`、隐藏点文件）。

### 4.4 权限与「一键放行」

默认对写文件、执行 shell 等会询问许可：

| 操作 | 说明 |
|------|------|
| `Ctrl+O` | 切换 always-approve（YOLO） |
| `/always-approve` | 同上 |
| `Shift+Tab` | 循环：Normal → Plan → Always-approve |
| `/auto` | 安全工具自动批、危险操作仍可能询问（需功能开启） |

只读类工具（读文件、列表、grep、部分只读 shell）通常不弹窗。

---

## 五、核心能力一览

| 能力 | 说明 |
|------|------|
| 读改文件 | 行级精确编辑，TUI 中显示 diff |
| 代码搜索 | ripgrep 驱动的 `grep` |
| Shell | 前台/后台命令、超时与输出截断可配 |
| 网络 | `web_search` / 抓取页面（可关） |
| 任务列表 | `todo_write`，`Ctrl+T` 切换 todos 面板 |
| 子代理 | `spawn_subagent`：探索 / 规划 / 通用并行 |
| Plan Mode | 先写计划再改代码 |
| Skills | 可复用的流程包，可变成 `/命令` |
| MCP | 外接 GitHub、数据库、Linear 等 |
| Hooks | 生命周期脚本（安全门禁、格式化、通知） |
| Plugins | 打包 skills/hooks/MCP/agents |
| Memory | 跨会话记忆（实验性） |
| 媒体 | `/imagine` 出图，`/imagine-video` 出视频 |
| 调度 | `/loop` 周期任务；monitor 流式事件 |
| 沙箱 | OS 级限制读写与子进程网络（Linux 更完整） |
| 无头模式 | 脚本与 CI |
| ACP Agent | IDE / 自定义客户端集成 |

---

## 六、键盘快捷键（实用子集）

完整表见官方 *Keyboard Shortcuts*；下列足够日常使用。

### 6.1 全局 / 会话

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+P` 或 `?` | 命令面板（快捷键、斜杠命令、skills） |
| `Ctrl+M` | 滚动区：模型选择器；输入区：多行模式切换 |
| `Ctrl+O` | 切换 always-approve |
| `Ctrl+S` | 会话选择器 / 恢复会话 |
| `Ctrl+N` | 新会话（需连按确认） |
| `Ctrl+Q` / `Ctrl+D` | 退出（VS Code 集成终端多用 `Ctrl+D`） |
| `Ctrl+G` | 把当前前台任务丢到后台 |
| `Ctrl+T` | Todos 面板 |
| `Ctrl+B` | Tasks 面板 |
| `Ctrl+;` | 提示词队列（若有排队消息） |
| `F2` 或 `Ctrl+,` | 设置 |
| `Shift+Tab` | 模式：Normal / Plan / Always-approve |

### 6.2 输入与中断

| 状态 | 按键 | 行为 |
|------|------|------|
| Agent 运行中 | `Ctrl+C` | 先清空草稿；空输入再按则取消当前 turn |
| Agent 运行中 | 纯 `Enter`（有文字） | **排队**后续消息 |
| Agent 运行中 | `Ctrl+Enter` / `Ctrl+I`（VS Code 系为 `Ctrl+L`） | **立即打断并发送** |
| 空闲、非空输入 | `Esc` `Esc`（800ms 内） | 清空输入 |
| 空闲、空输入且有历史 | `Esc` `Esc` | 打开 `/rewind` 时间线 |

> 注意：运行中 **`Esc` 不会取消任务**，取消请用 `Ctrl+C`。

### 6.3 回滚区导航

- **Simple 模式（默认）**：方向键选择条目；`Space` 回到输入；`Left`/`Right` 折叠/展开  
- **Vim 模式**（`/vim-mode` 或 `config.toml` 中 `[ui] vim_mode = true`）：`j`/`k`、`H`/`L`、折叠 `h`/`l`/`e`，复制 `y`/`Y` 等  

### 6.4 Shell 模式与历史

- 空输入时敲 `!` 进入 **shell 模式**（直接跑命令）  
- 空输入时 `↑` 打开提示历史；`/history` 可模糊搜索  

### 6.5 图片

支持拖入图片、粘贴截图（Windows 上截图粘贴常用 `Alt+V`）。非图片文件粘贴会变成绝对路径文本。

---

## 七、斜杠命令（Slash Commands）

输入 `/` 即可模糊补全。命令来源：内置、Skills、Plugins。

### 7.1 会话

| 命令 | 作用 |
|------|------|
| `/new`（`/clear`） | 新会话 |
| `/resume` | 打开会话选择器 |
| `/compact [说明]` | 压缩上下文；上下文约 **85%** 会自动 compact（可配） |
| `/context` | 上下文占用分解 |
| `/session-info` | 模型、turn 数、用量等 |
| `/fork` | 从当前点分叉新会话（可选 worktree） |
| `/rewind` | 回退到更早 turn，并尽量恢复文件快照 |
| `/copy [n]` | 复制最近（第 n 条）回复 |
| `/export` | 导出对话 |
| `/rename` / `/title` | 重命名会话 |
| `/home` / `/welcome` | 回欢迎页 |
| `/quit` / `/exit` | 退出 |

### 7.2 模型与模式

| 命令 | 作用 |
|------|------|
| `/model` / `/m` | 切换模型，如 `/model grok-build` |
| `/effort low\|medium\|high\|xhigh` | 推理强度（模型支持时） |
| `/always-approve` / `/auto` | 权限模式切换 |
| `/plan` / `/view-plan` | 进入 Plan / 查看计划 |
| `/multiline` | 多行输入 |
| `/vim-mode` | Vim 滚动导航 |
| `/minimal` / `/fullscreen` | 原生 scrollback 模式 ↔ 全屏 TUI |
| `/compact-mode` | 更紧凑的 UI 间距 |

### 7.3 记忆（需开启实验性 Memory）

| 命令 | 作用 |
|------|------|
| `/memory` / `/mem` | 浏览记忆；`on`/`off` 开关 |
| `/flush` | 立刻把本会话要点写入记忆 |
| `/dream` | 记忆整理合并 |
| `/remember ...` | 手动记一条（无需开 memory 实验特性也可记） |

启用方式：`grok --experimental-memory` 或 `GROK_MEMORY=1` 或 config `[memory] enabled = true`。

### 7.4 扩展与媒体

| 命令 | 作用 |
|------|------|
| `/plugins` `/hooks` `/marketplace` `/skills` `/mcps` | 扩展模态各页签 |
| `/imagine <描述>` | 文生图 |
| `/imagine-video <描述>` | 文/图生视频流程 |
| `/loop [间隔] <提示词>` | 周期任务，如 `/loop 30m 检查部署状态` |
| `/goal ...` | 跨 turn 的目标（功能开启时） |
| `/theme` | 主题 |
| `/settings` | 设置 UI |
| `/docs` | 内置 How-to / 打开在线文档 |
| `/terminal-setup` | 终端能力检测（真彩、剪贴板、tmux 等） |
| `/import-claude` | 导入 Claude Code 的权限、MCP、hooks 等 |
| `/login` `/logout` `/usage` `/privacy` | 账号与账单相关 |
| `/feedback` | 反馈 |
| `/btw ...` | 旁注，不打断当前主任务语义上的「插一句」 |
| `/config-agents` `/personas` | 管理 agent 定义与 personas |
| `/release-notes` | 更新日志 |

Skills 安装后也会出现在 `/` 菜单；重名时用限定名：`/user:commit`、`/local:commit`。

---

## 八、配置体系

### 8.1 优先级

1. CLI 参数  
2. 环境变量  
3. `~/.grok/config.toml`  
4. 组织 managed / requirements 配置  
5. 内置默认  

### 8.2 主配置 `~/.grok/config.toml`（常用片段）

```toml
[cli]
auto_update = true

[models]
default = "grok-build"
# web_search = "grok-4.20-multi-agent"  # 搜索工具可用的模型

[ui]
theme = "groknight"          # 或 auto / tokyonight / rosepine 等
simple_mode = true           # 输入区 readline；false 为实验性 vim 编辑
vim_mode = false             # 滚动区 vim 键
screen_mode = "fullscreen"   # 或 "minimal"
show_thinking_blocks = true
group_tool_verbs = true
collapsed_edit_blocks = false
remember_tool_approvals = false

[session]
auto_compact_threshold_percent = 85
load_envrc = true

[features]
telemetry = false
codebase_indexing = true
lsp_tools = false

[tools]
respect_gitignore = false

[toolset.bash]
timeout_secs = 120.0
output_byte_limit = 20000

[subagents]
enabled = true
```

外观细节（padding、滚动跟随、thinking 动画等）在 **`~/.grok/pager.toml`**。

### 8.3 项目级 `.grok/`

| 路径 | 用途 |
|------|------|
| `.grok/config.toml` | 项目 MCP、plugins、permission 规则、`[mcp] max_output_bytes` 等 |
| `.grok/skills/` | 项目 skills |
| `.grok/hooks/` | 项目 hooks |
| `.grok/agents/` | 项目 agent 定义 |
| `.grok/plugins/` | 项目插件 |
| `.grok/lsp.json` | 语言服务器 |
| `.grok/sandbox.toml` | 自定义沙箱配置文件 |
| `AGENTS.md` 等 | 项目规则（见下节） |

### 8.4 环境变量速查

| 变量 | 含义 |
|------|------|
| `XAI_API_KEY` | API Key |
| `GROK_HOME` | 配置根目录（默认 `~/.grok`） |
| `GROK_MEMORY` | `1`/`0` 开关记忆 |
| `GROK_SUBAGENTS` | 子代理开关 |
| `GROK_SANDBOX` | 沙箱 profile |
| `GROK_WEB_FETCH` | 网页抓取开关 |
| `GROK_LOG_FILE` / `RUST_LOG` | 调试日志 |

### 8.5 诊断

```bash
grok inspect          # 人类可读：skills、MCP、compat 等
grok inspect --json
grok models           # 可用模型列表
grok mcp list
grok mcp doctor
```

---

## 九、项目规则（AGENTS.md）

把「这个仓库怎么写代码」写进仓库，避免每会话重复说明。

**识别的文件名（同目录可多个）：**  
`Agents.md`、`Claude.md`、`CLAUDE.md`、`CLAUDE.local.md`、`AGENT.md`、`AGENTS.md`  

**规则目录：** `.grok/rules/`、以及兼容的 `.claude/rules/`、`.cursor/rules/`  

**发现顺序：** 全局 `~/.grok/` → 从仓库根到当前目录的每一层。**更深层优先**（后出现的指令覆盖冲突项）。

示例：

```markdown
# AGENTS.md

## 构建
- 使用 `pnpm`，不要用 npm。
- 提交前跑 `pnpm lint && pnpm test`。

## 风格
- TypeScript 严格模式；优先函数式组件。
- 不要擅自改无关文件。
```

全局规则可放在 `~/.grok/AGENTS.md`。

---

## 十、Skills：可复用工作流

Skill = 带 YAML frontmatter 的 `SKILL.md` 目录，描述「何时用、怎么做」。

### 10.1 存放位置（高优先级覆盖低优先级）

- `./.grok/skills/`（当前目录）  
- `<repo>/.grok/skills/`  
- `~/.grok/skills/`  
- 兼容 `~/.claude/skills/`、`./.claude/skills/`、Cursor skills 等  

### 10.2 最小示例

```markdown
---
name: commit
description: 按约定式提交规范写 commit。用户要提交或说 /commit 时使用。
---

# Commit

1. 查看 `git status` 与 `git diff`
2.  staged 后写清楚的 commit message
3. 执行 commit（不要 force push）
```

### 10.3 使用方式

- 斜杠：`/commit 修复登录超时`  
- 模型根据 `description` **自动触发**  
- `disable-model-invocation: true` 可禁止自动触发，只允许手动 `/`  
- 交互创建：`/create-skill`  

额外路径：

```toml
[skills]
paths = ["~/my-team-skills"]
disabled = ["wip-skill"]
```

---

## 十一、MCP：接外部世界

MCP（Model Context Protocol）把第三方工具暴露给模型。

### 11.1 配置示例

```toml
# ~/.grok/config.toml 或 项目 .grok/config.toml

[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { GITHUB_PERSONAL_ACCESS_TOKEN = "ghp_xxx" }
enabled = true
startup_timeout_sec = 30

[mcp_servers.remote]
url = "https://mcp.example.com/api/mcp"
headers = { "Authorization" = "Bearer token" }
```

### 11.2 CLI 管理

```bash
grok mcp list
grok mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem /path
grok mcp add --transport http sentry https://mcp.sentry.dev/mcp
grok mcp remove github
grok mcp doctor
```

`--scope project` 可写入项目配置，便于团队共享（密钥建议用 `${ENV}` 引用，勿硬编码）。

TUI 中：`/mcps` 管理已配置服务器。

---

## 十二、Plugins 与 Marketplace

**Plugin** 可打包：skills、commands、agents、hooks、MCP、LSP。

```bash
grok plugin list
grok plugin install owner/repo --trust
grok plugin uninstall <name>
```

TUI：`/plugins`、`/marketplace`。  
非 VS Code 系终端可用 `Ctrl+L` 打开扩展模态；VS Code 系请用 `/plugins`（`Ctrl+L` 被用作「立即打断发送」）。

---

## 十三、Hooks：生命周期自动化

在 Session 启动、用户提交、工具前后、压缩、结束等节点跑脚本或 HTTP。

**典型用途：** 拦截危险命令、审计日志、编辑后自动 format、发通知。

最小例子 `~/.grok/hooks/session-start.json`：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          { "type": "command", "command": "echo Grok session started in $(pwd)" }
        ]
      }
    ]
  }
}
```

**可拦截（blocking）的事件：** 主要是 `PreToolUse`（可 deny）。  
项目级 hooks 需 **folder trust**（首次信任仓库后才执行），避免不可信仓库跑任意代码。用 `/hooks` 查看与管理。

兼容 Claude / Cursor 的 hooks 配置（可在 `[compat.*]` 关闭扫描）。

---

## 十四、子代理（Subagents）与 Personas

主会话上下文有限时，可把「调研 / 写方案 / 实现」拆给子会话。

| 内置类型 | 特点 |
|----------|------|
| `general-purpose` | 通用，可写文件 |
| `explore` | 只读探索代码库 |
| `plan` | 只读出实现计划 |

- 默认开启；关闭：`GROK_SUBAGENTS=0` 或 `[subagents] enabled = false`  
- 可指定不同模型：`[subagents.models] explore = "grok-build"`  
- **Persona**：给子代理叠加语气/输出契约（`/personas`、`[subagents.personas.*]`）  
- **Agent 定义**：整会话级配置模型与工具集（`/config-agents`、`.grok/agents/`）  
- 子代理可 `background: true`，主会话稍后取结果  
- 隔离：`worktree` 模式在独立 git worktree 改代码，不污染主工作区  

主会话通过工具 `spawn_subagent` 启动；你侧主要感知是并行任务与汇总回报。

---

## 十五、Plan Mode：先计划后写代码

适合「做法不唯一、做错代价大」的任务（鉴权方案、缓存架构、大规模重构）。

**进入：**

- `/plan` 或 `/plan 迁移鉴权到新 API`  
- `Shift+Tab` 切到 Plan  
- 模型也可在歧义大时请求进入（需你批准）

**约束：** 除会话目录下的 `plan.md` 外，**拒绝其它文件编辑**（即使 always-approve 也不写业务代码）。

**审批界面：**

| 键 | 动作 |
|----|------|
| `a` | 批准并开始实现 |
| `s` | 要求修改计划 |
| `c` | 对选中行批注 |
| `q` | 放弃计划 |

`/view-plan` 可再次打开已保存计划。

---

## 十六、会话（Sessions）

每次对话自动落盘到 `~/.grok/sessions/<编码后的 cwd>/<session-id>/`，包含：

- 对话与工具流  
- 任务列表  
- rewind 文件快照  
- token 与统计  
- 子代理元数据  

**常用操作：** `/resume`、`grok -c`、`grok --resume <id>`、`/fork`、`/rewind`、`/compact`、`/rename`。

欢迎页与 `Ctrl+S` 也能挑历史会话。标题可搜索，会话内容也支持扩展搜索。

---

## 十七、跨会话 Memory（实验）

开启后，Grok 可检索以往会话中沉淀的事实与约定。

```bash
grok --experimental-memory
# 或
export GROK_MEMORY=1
```

存储大致在 `~/.grok/memory/`（全局 `MEMORY.md` + 按仓库区分的目录），支持 FTS 与可选向量检索。  
`/flush`、`/dream`、`/remember` 用于主动维护。`--no-memory` 强制关闭，优先级最高。

---

## 十八、无头模式（Headless）与脚本

```bash
# 纯文本结果
grok -p "用三句话总结这个仓库在做什么"

# JSON，便于管道
grok -p "Review changes for bugs" --output-format json --yolo | jq -r '.text'

# 流式 NDJSON
grok -p "..." --output-format streaming-json

# 限制工具
grok -p "解释架构" --tools "read_file,grep,list_dir"

# 去掉危险能力
grok -p "代码审查" --disallowed-tools "run_terminal_cmd,search_replace"

# 禁止子代理
grok -p "修 bug" --disallowed-tools "Agent"

# 权限规则（deny 优先）
grok -p "清理" --deny "Bash(rm*)" --allow "Bash(npm*)"

# 最大 agent turn 数
grok -p "..." --max-turns 20

# 结构化输出（配合 schema）
grok -p "..." --json-schema '{"type":"object","properties":{"ok":{"type":"boolean"}}}'
```

| 输出格式 | 标志 | 用途 |
|----------|------|------|
| plain | 默认 | 人读 |
| json | `--output-format json` | 单次结构化结果 |
| streaming-json | `streaming-json` | 实时事件流 |

无头模式下若工具需要交互批准，通常会取消并反馈给模型——自动化场景请用 `--yolo` / 明确的 allow 规则，或 `dontAsk` 策略（见权限文档）。

其它有用标志：`--check`（自检闭环）、`--best-of-n N`（多路并行择优）、`--rules "..."`、`--sandbox workspace`。

---

## 十九、Agent 模式与 IDE（ACP）

用于编辑器与自定义客户端，走 [Agent Client Protocol](https://agentclientprotocol.com)：

```bash
# stdio：IDE 扩展常用
grok agent stdio

# 本机 WebSocket 服务
grok agent serve --bind 127.0.0.1:2419 --secret <token>

# 连到中继（浏览器侧 UI）
grok agent headless --grok-ws-url wss://your-relay.example.com/ws
```

支持会话创建/加载、流式更新、思考流、权限交互等。客户端包括 Zed、Neovim、Emacs 扩展及自建工具。

---

## 二十、沙箱（Sandbox）

基于 OS 内核能力限制进程的文件与网络访问（Linux Landlock + seccomp 等；macOS Seatbelt）。**默认关闭。**

```bash
grok --sandbox workspace   # 推荐日常：可读全局，只写 CWD / ~/.grok / 临时目录
grok --sandbox read-only   # 只分析，不改项目文件
grok --sandbox strict      # 更严，适合不信任的代码
```

| Profile | 读 | 写 | 子进程网络（Linux） |
|---------|----|----|---------------------|
| off | 无限制 | 无限制 | 无限制 |
| workspace | 全局 | CWD + `~/.grok` + tmp | 允许 |
| read-only | 全局 | 仅 `~/.grok` + tmp | 阻止 |
| strict | 主要为 CWD + 系统路径 | CWD + `~/.grok` + tmp | 阻止 |

自定义 profile 写在 `~/.grok/sandbox.toml` 或 `.grok/sandbox.toml`，可用 `deny = ["**/.env", "**/*.pem"]` 等内核级拦截。

---

## 二十一、后台任务、监控与循环

- **后台 shell：** 工具带 `background: true`；`Ctrl+G` 也可把前台命令挪到后台  
- **取结果 / 等待 / 杀掉：** 对应 get / wait / kill 工具；Tasks 面板 `Ctrl+B`  
- **`/loop`：** 周期执行提示词，间隔最小 60 秒，默认 7 天后过期，最多约 50 个调度任务  
  ```text
  /loop 5m 检查测试是否全绿并汇报失败
  /loop 1h 总结自上次以来的新 commit
  ```  
- **monitor 工具：** 对长驻脚本的 stdout 按行推事件（适合日志 / CI 流）  

---

## 二十二、权限与安全（建议组合）

授权大致顺序：**PreToolUse hooks → deny/ask/allow 规则 → 记忆的批准 → 内置只读自动放行 → 当前 permission mode**。

| 模式 | 行为 |
|------|------|
| default | 非常规操作询问 |
| acceptEdits | 编辑类自动批 |
| bypassPermissions / always-approve | 尽量自动批（deny / 部分 ask 仍生效） |
| dontAsk | 无明确允许则拒绝（适合 CI） |

规则示例（CLI 可重复）：

```bash
--allow "Bash(npm*)" --deny "Bash(sudo*)" --deny "Edit(**/.env)"
```

**实践建议：**

1. 日常：`default` + 必要时 `Ctrl+O`  
2. 不信任代码：`--sandbox strict` + 不要 yolo  
3. CI：`dontAsk` + 白名单 `--allow` + 收窄 `--tools`  
4. 密钥：hooks deny 敏感路径；sandbox `deny` 配 `**/*.pem`  
5. 团队仓库：项目 hooks 走 trust；MCP 密钥用环境变量  

---

## 二十三、主题与终端

内置主题示例：GrokNight（默认暗色）、GrokDay、TokyoNight、RosePineMoon、OscuraMidnight；`theme = "auto"` 跟随系统明暗。

```text
/theme tokyonight
```

`/terminal-setup` 可查看真彩、剪贴板、tmux 透传等问题。  
**tmux** 通知常需：`set -g allow-passthrough on`。  
**minimal 模式**使用终端原生配色，不套主题。

---

## 二十四、自定义模型（BYOK / 兼容端点）

可在 config 里挂 OpenAI 兼容 / Anthropic Messages 等后端：

```toml
[model.my-proxy]
model = "grok-build"
base_url = "https://grok-proxy.acme.com/v1"
name = "Company Grok"
env_key = "XAI_API_KEY"
api_backend = "chat_completions"   # 或 responses / messages
context_window = 128000
```

`grok models` 查看列表；会话中 `/model` 或 `Ctrl+M` 切换。

---

## 二十五、推荐工作流

### 25.1 日常修 bug

```bash
cd your-repo
grok
```

提示词示例：

```text
复现 tests/auth 失败，修最小 diff，跑相关测试，最后说明原因。不要改无关文件。
```

### 25.2 大功能：先 Plan

```text
/plan 为 API 增加可选 Redis 缓存，兼容现有中间件
```

批准计划后再实现；用 `@` 钉住关键文件。

### 25.3 并行调研

```text
用 explore 子代理梳理鉴权相关调用链，主会话根据结论改代码。
```

### 25.4 团队标准化

1. 根目录 `AGENTS.md`：构建、风格、禁止事项  
2. `.grok/skills/`：发布、code review、commit 规范  
3. `.grok/config.toml`：共享 MCP（密钥用 env）  
4. hooks：PreToolUse 挡 `rm -rf`、强制 format  

### 25.5 CI 审查

```bash
grok -p "Review the diff for security and correctness. Output findings as a bullet list." \
  --output-format json \
  --disallowed-tools "search_replace" \
  --max-turns 15 \
  --yolo
```

### 25.6 与 Claude / Cursor 并存

默认会扫描 Claude、Cursor 的 skills / rules / MCP / hooks（可在 `[compat.*]` 关掉）。  
`/import-claude` 可批量导入本地 `~/.claude` 设置。

---

## 二十六、重要路径速查

| 路径 | 内容 |
|------|------|
| `~/.grok/config.toml` | 主配置 |
| `~/.grok/pager.toml` | TUI 外观 |
| `~/.grok/auth.json` | 登录凭证 |
| `~/.grok/sessions/` | 会话数据 |
| `~/.grok/memory/` | 跨会话记忆 |
| `~/.grok/skills/` | 用户 skills |
| `~/.grok/hooks/` | 用户 hooks |
| `~/.grok/plugins/` | 用户插件 |
| `~/.grok/agents/` | 用户 agent |
| `~/.grok/docs/user-guide/` | 本机完整文档 |
| `.grok/`（仓库内） | 项目级配置与扩展 |

---

## 二十七、常见问题

**Q：工具一直要确认？**  
用 `Ctrl+O` / `/always-approve`，或配置 `remember_tool_approvals` 与 allow 规则。

**Q：上下文爆了？**  
`/compact`；或提高 `[session] auto_compact_threshold_percent` 的触发频率（降低阈值）。大任务拆子代理。

**Q：想回到改坏之前？**  
`/rewind`（依赖会话内快照）；重要改动配合 git。

**Q：SSH 里无法浏览器登录？**  
`grok login --device-auth` 或 API Key。

**Q：VS Code 终端快捷键怪？**  
`Ctrl+L` 是「立即发送」；扩展用 `/plugins`；退出用 `Ctrl+D`。跑 `/terminal-setup`。

**Q：子模块 / monorepo 规则不生效？**  
确认在正确 cwd 启动；`AGENTS.md` 从 git root 到 cwd 会层层加载。

---

## 二十八、小结

Grok Build 把「聊天模型 + 本地工具链 + 会话持久化 + 可扩展生态」收成一个终端应用：

1. **装好登录** → `grok` 进仓库说话  
2. **用 `AGENTS.md` + Skills** 固化团队习惯  
3. **MCP / Plugins / Hooks** 接到真实工作流  
4. **Plan + Subagents + Sandbox** 处理大任务与风险  
5. **Headless / ACP** 接到 CI 与 IDE  

更细的键盘表、规则匹配语义、OTEL 监控等，建议在 TUI 内执行：

```text
/docs
/docs Getting Started
```

或打开在线文档：<https://docs.x.ai/build/overview>。

---

*若本文有与你本机版本不一致处，以 `grok --version` 对应文档与 `/release-notes` 为准。*
