---
title: "Pi Coding Agent 使用教程：最小内核、极强扩展的终端 AI 编程助手"
date: 2026-08-02T22:00:00+08:00
draft: false
tags: ["工具链", "AI", "Agent", "Pi"]
description: "Pi 是一个开源终端 AI 编码 Agent，遵循'提供原语而非功能'的设计哲学。本文讲解其安装、模型配置、四种运行模式、会话管理、扩展与技能开发，以及安全模型。"
---

> **说明：** 本文参考菜鸟教程《Pi Coding Agent 入门教程》（runoob.com/ai-agent/pi-coding-agent.html）与官方文档（pi.dev/docs/latest）整理。撰写时 Pi 版本约为 `0.80.6`，不同版本命令可能有差异，以 `pi --help` 为准。  
> 本文假设你用过至少一种终端 AI 编程助手（如 Claude Code、Codex CLI），对 agent 能读写文件、执行命令、调用工具有基本理解。

## 一、Pi 是什么

Pi（全称 Pi Coding Agent）是一个运行在终端里的 AI 编码 Agent（agent harness），由 earendil-works 开发，GitHub 地址：github.com/earendil-works/pi。

与 Claude Code、OpenAI Codex CLI、Grok Build 同类产品，Pi 的核心差异在于设计哲学。Pi 官方将其表述为：

> "There are many agent harnesses, but this one is yours."
>（Agent 工具很多，但这个是你自己的。）

Pi 不追求内置大量功能，而是提供一组最小化的"原语"（primitive，即基础构建单元），让你按需组装、扩展、甚至让 Pi 自己编写扩展。

**四个核心包：**

| 包名 | 作用 | 何时单独用到 |
|------|------|--------------|
| `@earendil-works/pi-coding-agent` | 交互式编码 Agent CLI，用户直接安装的入口 | 绝大多数情况只需这个 |
| `@earendil-works/pi-agent-core` | Agent 运行时，负责工具调用循环和状态管理 | 自建非编码类 Agent |
| `@earendil-works/pi-ai` | 统一的多厂商大模型 API（OpenAI、Anthropic、Google 等） | 只用统一接口调多家模型 |
| `@earendil-works/pi-tui` | 终端 UI 库，支持差分渲染（只重绘变化的屏幕区域） | 自建终端应用 |

日常使用时，你只需要 `npm install -g @earendil-works/pi-coding-agent`，其余三个包作为依赖自动安装。

---

## 二、安装与配置

### 2.1 安装

**方式一：一键脚本（推荐）**

macOS 和 Linux：

```bash
curl -fsSL https://pi.dev/install.sh | sh
```

Windows PowerShell：

```powershell
powershell -c "irm https://pi.dev/install.ps1 | iex"
```

**方式二：包管理器全局安装**

```bash
# npm
npm install -g --ignore-scripts @earendil-works/pi-coding-agent

# pnpm
pnpm add -g --ignore-scripts @earendil-works/pi-coding-agent

# bun
bun add -g --ignore-scripts @earendil-works/pi-coding-agent
```

`--ignore-scripts` 参数跳过依赖的安装期生命周期脚本。Pi 正常运行不需要这些脚本，加上可减少潜在的安全风险。

**验证安装：**

```bash
pi --version
```

预期输出示例：

```
0.80.6
```

### 2.2 卸载

```bash
npm uninstall -g @earendil-works/pi-coding-agent
# 或 pnpm remove -g ...
# 或 bun uninstall -g ...
```

**注意：** 卸载 Pi 不会删除配置。设置、凭证、会话记录、已安装的扩展都保留在 `~/.pi/agent/` 目录里。如需彻底清理，手动删除该目录。

---

## 三、模型供应商配置

Pi 本身不包含大模型，需要你提供至少一个模型供应商的访问凭证。

### 3.1 方式一：订阅登录

如果你已有 Claude Pro/Max、ChatGPT Plus/Pro 或 GitHub Copilot 订阅，可以直接登录复用：

```bash
pi        # 启动 Pi
/login    # 在 Pi 内执行登录命令
```

可选供应商：

| 供应商 | 要求 | 说明 |
|--------|------|------|
| Claude Pro / Max | Anthropic 订阅 | 用量按 token 计费，不占订阅额度 |
| ChatGPT Plus / Pro | OpenAI 订阅 | 通过 Codex for OSS 认可 |
| GitHub Copilot | Copilot 订阅 | 若提示模型不支持，需在 VS Code 中先启用 |

凭证存入 `~/.pi/agent/auth.json`，过期自动刷新。退出登录用 `/logout`。

### 3.2 方式二：API Key

通过环境变量传入，以 Anthropic 为例：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

或在 Pi 内运行 `/login` 选择 API Key 类型供应商，将 Key 持久化存入 `auth.json`。

**常见供应商的环境变量：**

| 供应商 | 环境变量 | auth.json 键名 |
|--------|----------|----------------|
| Anthropic | `ANTHROPIC_API_KEY` | `anthropic` |
| OpenAI | `OPENAI_API_KEY` | `openai` |
| Google Gemini | `GEMINI_API_KEY` | `google` |
| DeepSeek | `DEEPSEEK_API_KEY` | `deepseek` |
| Groq | `GROQ_API_KEY` | `groq` |
| Mistral | `MISTRAL_API_KEY` | `mistral` |
| xAI | `XAI_API_KEY` | `xai` |
| OpenRouter | `OPENROUTER_API_KEY` | `openrouter` |
| Hugging Face | `HF_TOKEN` | `huggingface` |

**凭证优先级（从高到低）：**

1. 命令行 `--api-key` 参数
2. `auth.json` 里的条目
3. 环境变量
4. `models.json` 自定义供应商里的 Key

`auth.json` 创建时权限设为 `0600`（仅当前用户可读写）。

### 3.3 自定义供应商配置（以 DeepSeek 为例）

Pi 通过 `models.json` 支持自定义供应商。配置文件路径：

- Linux / macOS：`~/.pi/agent/models.json`
- Windows：`%USERPROFILE%\.pi\agent\models.json`

示例（DeepSeek V4 Pro / V4 Flash）：

```json
{
  "providers": {
    "deepseek": {
      "baseUrl": "https://api.deepseek.com",
      "api": "openai-completions",
      "apiKey": "$DEEPSEEK_API_KEY",
      "models": [
        {
          "id": "deepseek-v4-pro",
          "name": "DeepSeek V4 Pro",
          "contextWindow": 1000000,
          "maxTokens": 384000,
          "input": ["text"],
          "reasoning": true,
          "cost": {
            "input": 1.74,
            "output": 3.48,
            "cacheRead": 0.145,
            "cacheWrite": 0
          },
          "compat": {
            "requiresReasoningContentOnAssistantMessages": true,
            "thinkingFormat": "deepseek"
          }
        }
      ]
    }
  }
}
```

设置环境变量后启动 Pi，输入 `/model` 打开模型选择器即可切换。

---

## 四、核心工作流

### 4.1 首次运行

```bash
cd /path/to/project
pi
```

启动后输入一句话并回车：

```
Summarize this repository and tell me how to run its checks.
```

Pi 会自动读取文件、运行命令、给出总结。

### 4.2 四个内置工具

Pi 默认只给模型四个工具：

| 工具 | 作用 | 默认启用 |
|------|------|----------|
| `read` | 读取文件内容 | 是 |
| `write` | 创建或覆盖文件 | 是 |
| `edit` | 对文件做局部补丁修改 | 是 |
| `bash` | 运行 shell 命令 | 是 |
| `grep` | 搜索文件内容 | 否 |
| `find` | 按条件查找文件 | 否 |
| `ls` | 列出目录内容 | 否 |

你可以通过命令行参数控制可用工具：

```bash
# 只读模式
pi --tools read,grep,find,ls -p "审查这段代码"

# 禁用某个工具
pi --exclude-tools ask_question

# 关闭所有内置工具
pi --no-builtin-tools -e ./my-extension.ts

# 完全禁用工具，纯对话
pi --no-tools -p "解释一下这个概念"
```

### 4.3 用 AGENTS.md 给项目下指令

在项目根目录创建 `AGENTS.md`（或 `CLAUDE.md`，Pi 两种文件名都识别）：

```markdown
# 项目指令

- 改完代码后运行 `npm run check`。
- 不要在本地跑生产环境的数据库迁移。
- 回答尽量简洁。
```

Pi 启动时从以下位置加载指令文件：

| 位置 | 作用范围 |
|------|----------|
| `~/.pi/agent/AGENTS.md` | 全局，所有项目生效 |
| 父目录的 `AGENTS.md` | 向上逐层查找 |
| 当前目录的 `AGENTS.md` | 最具体，优先级最高 |

修改指令文件后，在 Pi 内运行 `/reload` 或重启 Pi，新指令才会生效。

如需替换整个系统提示，在项目放 `.pi/SYSTEM.md`；追加则放 `.pi/APPEND_SYSTEM.md`。

### 4.4 项目信任

如果当前目录或上层目录存在 `.pi/settings.json`、`.pi/extensions`、`.pi/skills` 等项目级资源，Pi 交互式启动时会询问是否信任该项目。

| 信任级别 | 行为 | 设置方式 |
|----------|------|----------|
| `ask`（默认） | 交互式询问；非交互模式忽略项目资源 | `settings.json` 的 `defaultProjectTrust` |
| `always` | 始终信任 | 同上，或用 `-a` / `--approve` |
| `never` | 从不信任 | 同上，或用 `-na` / `--no-approve` |

信任决策保存在 `~/.pi/agent/trust.json`，按目录路径记录。

---

## 五、交互模式操作

### 5.1 快捷输入

| 操作 | 方式 | 说明 |
|------|------|------|
| 引用文件 | 输入 `@` 触发模糊搜索 | 文件内容作为上下文发给模型 |
| 运行命令 | `!npm run lint` | 输出进入模型上下文 |
| 运行但不进上下文 | `!!npm run build` | 两个感叹号 |
| 多行输入 | `Shift+Enter` | Windows Terminal 用 `Ctrl+Enter` |
| 粘贴图片 | `Ctrl+V` | 也支持拖拽进终端 |
| 复制回复 | `Ctrl+X` | 复制最后一条助手消息 |
| 外部编辑器 | `Ctrl+G` | 打开 `$VISUAL` / `$EDITOR` 编辑长文本 |

### 5.2 切换模型与思考强度

| 操作 | 快捷键 / 命令 | 说明 |
|------|---------------|------|
| 打开模型选择器 | `/model` 或 `Ctrl+L` | 从所有可用模型中选择 |
| 切换思考强度 | `Shift+Tab` | 在 off / minimal / low / medium / high / xhigh / max 之间循环 |
| 循环收藏模型 | `Ctrl+P` / `Shift+Ctrl+P` | 在预先圈定的模型间快速切换 |

启动时直接指定：

```bash
pi --provider openai --model gpt-4o "帮我重构"
pi --model sonnet:high "解决这个复杂问题"
pi --models "claude-*,gpt-4o"
```

### 5.3 边跑边插话

Agent 工作时，你可以随时插入消息：

| 按键 | 行为 | 适用场景 |
|------|------|----------|
| `Enter` | 转向消息：当前工具跑完后立刻送达 | 发现方向跑偏，及时纠正 |
| `Alt+Enter` | 后续消息：等 Agent 完全跑完再送达 | 想追加一个新需求 |
| `Escape` | 中止当前工作，把排队消息还原回编辑器 | 想停下来重新组织 |

### 5.4 斜杠命令速查

| 命令 | 功能 |
|------|------|
| `/login` `/logout` | 管理 OAuth 或 API Key 凭证 |
| `/model` | 切换模型 |
| `/scoped-models` | 设置哪些模型参与 `Ctrl+P` 循环 |
| `/resume` | 从历史会话选一个继续 |
| `/new` | 开新会话 |
| `/name <名称>` | 给当前会话起名 |
| `/tree` | 跳到会话树的任意节点继续 |
| `/fork` | 从某条历史消息分叉出新会话 |
| `/clone` | 复制当前分支成新会话 |
| `/compact [提示]` | 手动压缩上下文，可附带保留重点的指令 |
| `/export [文件]` | 导出会话为 HTML 或 JSONL |
| `/import <文件>` | 从 JSONL 导入并恢复会话 |
| `/share` | 上传为私有 GitHub Gist，生成分享链接 |
| `/reload` | 重载快捷键、扩展、技能、提示、上下文文件 |

### 5.5 会话管理

Pi 的会话不是线性历史，而是树形结构。从任意历史节点分叉不会丢失原有分支，所有分支存在同一个会话文件里。

会话自动保存到 `~/.pi/agent/sessions/`，按工作目录归类。

```bash
pi -c          # 继续最近一次会话
pi -r          # 浏览并选择历史会话
pi --name "任务名"  # 命名会话
```

当对话接近模型上下文上限时，Pi 会自动压缩早期消息为摘要。也可主动用 `/compact` 触发：

```
/compact 重点保留最近的改动和错误处理
```

---

## 六、四种运行模式

### 6.1 Interactive（交互式）

默认模式，完整终端 UI 体验，日常编码使用。

```bash
pi
```

### 6.2 Print（一次性输出）

用 `-p` 让 Pi 回答完就退出，适合脚本调用：

```bash
pi -p "总结一下这个代码库"

# 配合管道
cat README.md | pi -p "总结这段文字"

# 带图片提问
pi -p @screenshot.png "这张图里是什么？"
```

### 6.3 JSON（事件流）

`--mode json` 将所有事件以 JSON 行输出，方便程序解析：

```bash
pi --mode json -p "列出 src 下所有 .ts 文件"
```

### 6.4 RPC（进程间通信）

`--mode rpc` 通过 stdin/stdout 走 JSON 协议，适合非 Node 程序（如 Python、Go）驱动 Pi。

### 6.5 SDK（库模式）

作为库嵌入到你的 Node 应用中，比 RPC 更紧密的集成。详见官方文档 pi.dev/docs/latest/sdk。

---

## 七、扩展与技能

### 7.1 扩展（Extensions）

扩展是 TypeScript 模块，能订阅生命周期事件、注册自定义工具、添加命令和快捷键。通过 jiti 加载，无需编译。

**存放位置：**

| 位置 | 作用范围 |
|------|----------|
| `~/.pi/agent/extensions/*.ts` | 全局，所有项目生效 |
| `.pi/extensions/*.ts` | 项目级，仅当前项目生效 |

**最小扩展示例：**

```typescript
// ~/.pi/agent/extensions/my-extension.ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // 1. 会话启动时通知
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("扩展已加载！", "info");
  });

  // 2. 拦截危险命令
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("危险操作！", "允许执行 rm -rf 吗？");
      if (!ok) return { block: true, reason: "被用户阻止" };
    }
  });

  // 3. 注册自定义工具
  pi.registerTool({
    name: "greet",
    label: "打招呼",
    description: "按名字向某人打招呼",
    parameters: Type.Object({
      name: Type.String({ description: "要打招呼的名字" }),
    }),
    async execute(_toolCallId, params) {
      return {
        content: [{ type: "text", text: `你好，${params.name}！` }],
        details: {},
      };
    },
  });

  // 4. 注册斜杠命令 /hello
  pi.registerCommand("hello", {
    description: "打个招呼",
    handler: async (args, ctx) => {
      ctx.ui.notify(`你好 ${args || "world"}！`, "info");
    },
  });
}
```

用 `-e` 参数临时加载测试：

```bash
pi -e ./my-extension.ts
```

**扩展能力总览：**

| 能力 | 对应方法 | 典型用途 |
|------|----------|----------|
| 注册自定义工具 | `pi.registerTool()` | 让模型调用业务接口 |
| 注册命令 | `pi.registerCommand()` | 添加斜杠命令 |
| 注册快捷键 | `pi.registerShortcut()` | 绑定自定义快捷键 |
| 拦截工具调用 | `tool_call` 事件 | 权限确认、命令改写 |
| 修改上下文 | `context` 事件 | 过滤历史、注入 RAG 内容 |
| 注入消息 | `pi.sendMessage()` | 动态补充上下文 |

Pi 仓库里有 50 多个示例扩展，涵盖子 Agent、计划模式、权限门、SSH 执行、沙箱等场景。

### 7.2 技能（Skills）

技能是另一种可复用能力包，与扩展的区别在于：技能主要是给模型看的指令和脚本，按需加载。

Pi 实现了 Agent Skills 标准，也能直接使用 Claude Code、OpenAI Codex 等工具的技能。

**存放位置：**

| 位置 | 作用范围 |
|------|----------|
| `~/.pi/agent/skills/` | 全局 |
| `~/.agents/skills/` | 全局，跨工具共享 |
| `.pi/skills/` | 项目级（需项目被信任） |
| `.agents/skills/` | 项目级，跨工具共享 |

**技能目录结构：**

```
my-skill/
├── SKILL.md              # 必需：前置信息 + 指令
├── scripts/
│   └── process.sh
├── references/
│   └── api-reference.md
└── assets/
    └── template.json
```

**SKILL.md 写法：**

```markdown
---
name: my-skill
description: 处理 PDF 文件，提取文本和表格、填表单、合并多个 PDF。
---

# My Skill

## Setup

首次使用前运行：

​```bash
cd /path/to/skill && npm install
​```

## Usage

​```bash
./scripts/process.sh <input>
​```
```

技能采用渐进式披露（progressive disclosure）：启动时 Pi 只把每个技能的名字和描述放进系统提示，当任务匹配时模型才用 `read` 工具加载完整说明。这样既能装很多技能，又不会一开始就把上下文占满。

手动调用：

```
/skill:brave-search
/skill:pdf-tools extract
```

### 7.3 Pi 包管理

扩展、技能、提示模板、主题都可以打包成 Pi 包，通过 npm 或 git 分享安装。

```bash
# 从 npm 安装
pi install npm:@foo/pi-tools

# 从 git 仓库安装
pi install git:github.com/badlogic/pi-doom

# 项目本地安装
pi install npm:@foo/pi-tools -l

# 卸载
pi remove npm:@foo/pi-tools

# 列出已安装包
pi list

# 更新 Pi 和所有包
pi update --all
```

---

## 八、安全须知

### 8.1 没有内置沙箱

Pi 不包含内置沙箱。它以启动它的用户身份运行，拥有该用户的全部权限。内置工具可以读写文件、运行 shell 命令，扩展是 TypeScript 模块，权限与 Pi 进程相同。

这是有意为之：Pi 要在本机源码上工作、调用项目工具链、融入开发环境，一个不完整的进程内沙箱会让人误以为是安全边界，实际却仍依赖宿主 shell、文件系统和凭证。

真正的隔离必须来自操作系统或容器/虚拟化边界，而不是 Pi 内部。

### 8.2 项目信任

项目信任是"输入加载守卫"——它防止仓库静默改你的设置或扩展，但不能让不可信的代码、提示或模型输出变得安全。仓库文件、注释、文档里的提示注入（prompt injection）是本地 Agent 固有的风险，Pi 无法可靠阻止。

### 8.3 处理不可信或无人值守的任务

官方建议把 Pi 放进隔离环境里跑：

1. 把整个 `pi` 进程放进容器、虚拟机或远程沙箱
2. 只挂载 Agent 应该访问的工作区路径
3. 除非必要，不要挂载宿主机的 `~/.pi/agent`（含凭证和会话）
4. 只传最少必要的 API Key，或用短期凭证
5. 任务不需要联网时，限制网络访问
6. 把结果复制回可信系统前，先审查 diff 和输出

注意：如果以读写方式挂载宿主工作区，容器内的写入仍会改到宿主文件。更强的防范是用只读挂载，或把文件复制进出沙箱。

---

## 九、与其他工具对比

| 特性 | Pi | Claude Code | Codex CLI | Grok Build |
|------|----|-------------|-----------|------------|
| 开源 | 是（GitHub） | 否 | 部分开源 | 否 |
| 内置功能 | 极少（扩展实现） | 较多 | 中等 | 较多 |
| 扩展机制 | TypeScript 扩展 + Skills | Skills + MCP | Skills + Plugins | Skills + MCP |
| 自定义供应商 | `models.json` 直接写 | 有限 | 有限 | 有限 |
| 沙箱 | 无（需容器） | 有审批流 | 有审批流 | 有沙箱选项 |
| 多模型切换 | 15+ 供应商 | 主要 Anthropic | 主要 OpenAI | 多供应商 |
| 会话树 | 支持 | 线性为主 | 有限 | 支持 |

Pi 的优势在于开放性和可扩展性——你可以自己写扩展实现任何功能，不受产品路线图限制。代价是需要更多手动配置和安全意识。

---

## 十、快速上手流程

```bash
# 1. 安装
curl -fsSL https://pi.dev/install.sh | sh

# 2. 配置模型（以 Anthropic 为例）
export ANTHROPIC_API_KEY=sk-ant-...

# 3. 进入项目
cd /path/to/project

# 4. 启动
pi

# 5. 第一句话
> 分析这个项目结构，告诉我怎么运行测试。
```

启动后可以用 `/` 查看斜杠命令列表，`Ctrl+G` 打开外部编辑器写长消息，`@` 引用文件，`!` 运行命令。
