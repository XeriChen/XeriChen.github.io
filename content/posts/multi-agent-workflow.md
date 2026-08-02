---
title: "多 Agent 协同工作流实战：tmux、git worktree 与手动终端编排"
date: 2026-08-02T21:00:00+08:00
draft: false
tags: ["工具链", "AI", "Agent", "Workflow"]
description: "面向已用过单个 AI 编程助手的读者，讲解如何手动编排多个 agent 并行工作——tmux 分屏、git worktree 隔离、agent 间通信，以及实操终端命令。"
---

> **说明：** 本文写给已经用过至少一种终端 AI 编程助手（如 Claude Code、Codex CLI、Grok Build）的读者。文中假设你理解"agent 能读写文件、执行命令、调用工具"这一基本模型。如果需要补充单 agent 工作原理，可先读本站《LLM + Agent 原理：从聊天机器人到能干活的智能体》。  
> 文中命令基于 Linux/macOS 终端，Windows 用户建议在 WSL2 下操作。

## 一、为什么单个 agent 不够用

截至 2026 年 8 月，主流大语言模型的上下文窗口（context window，模型一次能处理的最大文本长度）普遍达到 100 万 token。这个数字足够装下一本中等厚度的书，但实际开发任务中，单 agent 仍会遇到三个结构性瓶颈。

**1. 注意力质量随上下文长度下降**

100 万 token 不等于模型能同等精度地利用其中每一处信息。多项评测显示，当上下文超过 20 万 token 后，模型对早期信息的检索准确率明显降低。一个需要处理整个代码仓库、阅读大量日志、维护对话历史的 agent，工作 2–3 轮后就会把早期的指令或发现"忘掉"。

**2. 串行执行浪费时间**

单 agent 同一时刻只能做一件事。假设任务需要：修复一个 bug、写一个测试、更新相关文档。串行执行时，每一步都要等前一步完成、上下文传回模型、模型生成下一步。如果三步可以并行（它们通常互不依赖），单 agent 会把本可以并行的 15 分钟变成 45 分钟。

**3. 工作区互相污染**

一个 agent 在自己的工作目录里改代码，另一个 agent 也在同一个目录里改代码，两者的改动可能冲突。即使不冲突，agent A 的中间产物（临时文件、未提交的编辑）可能干扰 agent B 对项目状态的判断。

**多 agent 的核心价值**：把大任务拆成互不依赖的子任务，每个 agent 在独立的目录、独立的终端里运行，由你（或一个"主管 agent"）负责汇总。并行 + 隔离 + 专业化。

---

## 二、tmux：终端里的多窗口管理器

tmux（terminal multiplexer，终端复用器）允许你在一个终端窗口里创建多个"窗格"（pane），每个窗格运行独立的 shell 会话。即使关闭终端窗口、断开 SSH，tmux 里的会话也不会中断。

### 2.1 核心概念

| 概念 | 说明 |
|------|------|
| Session（会话） | 最高层容器，可包含多个 Window。断开后重连用 |
| Window（窗口） | 类似浏览器的标签页，全屏显示一个 Pane（或一组 Pane） |
| Pane（窗格） | 终端的一个矩形区域，运行一个 shell 进程 |

一个典型布局：

```
┌─────────────────────────────┐
│ Session: "multi-agent"      │
│ ┌─────────────┬───────────┐ │
│ │ Window 0    │           │ │
│ │ Pane 0      │ Pane 1    │ │
│ │ (agent A)   │ (agent B) │ │
│ ├─────────────┴───────────┤ │
│ │ Window 1                │ │
│ │ Pane 0                  │ │
│ │ (monitor / summary)     │ │
│ └─────────────────────────────┘
```

### 2.2 安装

```bash
# Debian / Ubuntu
sudo apt install tmux

# macOS
brew install tmux

# 验证
tmux -V
```

### 2.3 常用命令速查

tmux 的操作依赖"前缀键"（prefix key），默认为 `Ctrl+b`。先按前缀键，再按功能键。以下列出文中用到的命令：

```bash
# 新建命名会话
tmux new-session -s multi-agent

# 水平分割当前窗格（上下排列）
Ctrl+b "

# 垂直分割当前窗格（左右排列）
Ctrl+b %

# 在窗格间移动焦点
Ctrl+b 方向键

# 放大当前窗格为全屏（再按一次恢复）
Ctrl+b z

# 断开会话（会话内进程继续运行）
Ctrl+b d

# 重新连接
tmux attach -t multi-agent

# 列出所有会话
tmux ls

# 在会话内切换窗口
Ctrl+b n    # 下一个窗口
Ctrl+b p    # 上一个窗口

# 关闭当前窗格（确认后）
exit
# 或 Ctrl+d
```

**注意**：如果你通过 SSH 远程操作，tmux 的优势更明显——断开 SSH 后会话不丢失，重新连上 `tmux attach` 即可恢复。

---

## 三、git worktree：让一个仓库同时检出多个分支

### 3.1 为什么分支不够

Git 的分支（branch）可以让你在不同版本间切换，但同一时刻只有一个分支处于"已检出"状态，即工作目录里的文件反映的是该分支的内容。

如果你需要同时查看或修改两个分支，通常的做法是：
1. 在分支 A 工作，提交或暂存（`git stash`）
2. 切到分支 B（`git checkout B`）
3. 工作完再切回去，恢复暂存

这种做法的问题：
- 切分支期间，分支 A 的中间状态必须被保存或提交，否则会丢
- 两个分支如果有未提交的互相冲突的修改，`git checkout` 会拒绝切换
- 每次切换都要等 git 重写工作目录文件，大项目可能有几秒延迟

**git worktree** 解决了这个问题：它允许你从同一个 git 仓库创建多个独立的"工作树"（worktree），每个工作树在不同的目录里，可以同时处于不同分支，互不干扰。

### 3.2 工作原理

一个 git 仓库（`.git` 目录）存储了所有历史记录。worktree 本质上是一个指向该仓库的"外部目录"，拥有自己的工作目录和当前分支，但共享仓库对象数据库。

```
project/                  <-- 主工作目录（main 分支）
├── .git/                 <-- 仓库元数据
├── src/
└── ...

project-auth/             <-- worktree 目录（feat-auth 分支）
├── src/                  <-- 独立的工作目录
└── .git                  <-- 指向主仓库的文件（gitlink）
```

两个目录的磁盘占用并不翻倍：历史记录共享，只多出各自工作目录的差异文件。

### 3.3 常用命令

```bash
# 列出当前所有 worktree
git worktree list

# 从 main 分支创建新 worktree，目录名为 project-auth
git worktree add ../project-auth feat-auth

# 如果分支不存在，基于当前 HEAD 新建分支并创建 worktree
git worktree add -b feat-auth ../project-auth

# 进入 worktree 目录工作
cd ../project-auth
git checkout feat-auth
# ... 编辑、提交 ...

# 删除 worktree（目录必须已清理，没有未提交的改动）
git worktree remove ../project-auth

# 清理已删除但 git 仍有记录的 worktree
git worktree prune
```

**重要限制**：同一个分支不能同时被两个 worktree 检出。如果你想让两个 agent 各自在独立目录里工作，必须给每个 agent 分配一个独立分支。

### 3.4 适用场景

| 场景 | 是否需要 worktree |
|------|-------------------|
| 两个 agent 改同一个分支的不同文件 | 需要 |
| 两个 agent 分别开发独立功能（不同分支） | 需要 |
| 一个 agent 改代码，一个 agent 只读代码审查 | 需要（审查 agent 检出稳定分支） |
| 两个 agent 只读、不改文件 | 可以共享同一工作目录，但建议隔离避免日志文件冲突 |

---

## 四、手动编排：从零启动三个并行 agent

下面给出一个完整可运行的终端操作示例。任务设定：你有一个 Web 应用项目，需要同时完成三件事——实现用户认证模块（feat-auth）、写单元测试（fix-tests）、更新 API 文档（update-docs）。三个子任务互不依赖，可以并行。

### 4.1 第一步：准备工作

```bash
# 进入项目目录
cd ~/projects/myapp

# 确认当前在 main 分支，工作区干净
git status
# 输出应为：On branch main / nothing to commit, working tree clean

# 给每个子任务创建独立分支
git branch feat-auth
git branch fix-tests
git branch update-docs

# 为每个分支创建 worktree
git worktree add ../myapp-auth feat-auth
git worktree add ../myapp-tests fix-tests
git worktree add ../myapp-docs update-docs

# 确认三个 worktree 已就绪
git worktree list
# 输出应包含三个路径，分别指向三个分支
```

### 4.2 第二步：启动 tmux 会话

```bash
# 创建名为 "dev" 的 tmux 会话，第一个窗格自动创建
tmux new-session -s dev

# 当前在 Pane 0。垂直分割，右侧创建 Pane 1
Ctrl+b %

# 在 Pane 1 中再水平分割，下方创建 Pane 2
Ctrl+b "

# 现在有三个窗格：Pane 0（左）、Pane 1（右上）、Pane 2（右下）
```

### 4.3 第三步：在每个窗格启动 agent

**Pane 0（实现认证模块）：**

```bash
cd ~/projects/myapp-auth
claude "实现用户认证模块：注册、登录、登出。使用 JWT，密码 bcrypt 哈希。完成后提交到 feat-auth 分支。"
```

**Pane 1（写单元测试）：**

```bash
cd ~/projects/myapp-tests
claude "为 src/utils/ 下的所有工具函数补充单元测试。使用项目现有的测试框架。完成后提交到 fix-tests 分支。"
```

**Pane 2（更新文档）：**

```bash
cd ~/projects/myapp-docs
claude "更新 docs/api.md，根据当前 src/routes/ 下的路由代码同步 API 文档。完成后提交到 update-docs 分支。"
```

三个 agent 开始并行工作。你可以用 `Ctrl+b 方向键` 在窗格间切换，观察每个 agent 的进度。

### 4.4 第四步：监控与干预

当你发现某个 agent 输出偏离预期时，可以：

1. **暂停 agent**：在对应窗格按 `Ctrl+C`（大多数终端 agent 会中断当前操作）
2. **追加指令**：直接输入补充说明，例如"不要用 JWT，改用 session"
3. **恢复**：agent 通常会继续对话上下文

如果 agent 陷入循环（反复执行相同命令），`Ctrl+C` 后输入："停止当前操作，总结已完成的内容，等待新指令。"

### 4.5 第五步：收尾

三个 agent 都完成后：

```bash
# 回到主工作目录
cd ~/projects/myapp

# 合并三个分支到 main
git merge feat-auth fix-tests update-docs

# 如有冲突，手动解决后
git add .
git commit -m "merge: 集成认证模块、测试、文档更新"

# 清理 worktree（确认各分支已合并）
git worktree remove ../myapp-auth
git worktree remove ../myapp-tests
git worktree remove ../myapp-docs

# 删除已合并的分支（可选）
git branch -d feat-auth fix-tests update-docs

# 断开 tmux 会话
tmux kill-session -s dev
```

---

## 五、Agent 间通信：文件约定与同步原则

多个 agent 并行工作时，需要一个机制让它们交换信息。最可靠的方式是**基于文件的约定**——每个 agent 按约定格式读写特定文件，其他 agent 按需读取。

### 5.1 文件结构示例

在项目根目录放置一个共享目录：

```
myapp/
├── agents/
│   ├── TODO.md          <-- 任务总表，所有 agent 可读可写
│   ├── handoff/         <-- agent 间传递消息的目录
│   │   ├── auth-done.md
│   │   ├── tests-done.md
│   │   └── docs-done.md
│   └── context.md       <-- 当前项目状态摘要（只读，由主管更新）
```

**TODO.md 格式：**

```markdown
## 任务总表

- [x] 实现认证模块 → agent-auth（完成于 2026-08-02 15:30）
- [ ] 编写单元测试 → agent-tests（进行中）
- [ ] 更新 API 文档 → agent-docs（进行中）
```

**handoff/auth-done.md 格式：**

```markdown
## 完成通知

完成时间：2026-08-02 15:30
分支：feat-auth
变更文件：
- src/auth/login.go（新增）
- src/auth/register.go（新增）
- src/middleware/auth.go（修改）

接口说明：
- POST /api/login → 返回 JWT token
- POST /api/register → 返回用户 ID
```

### 5.2 三条避冲突原则

**原则一：写自己的文件，读别人的文件**

每个 agent 只能修改自己负责的文件（如 `handoff/auth-done.md`），但可以自由读取其他 agent 的输出文件。这样即使两个 agent 同时写入，也不会互相覆盖。

**原则二：通过 git 同步，不通过网络**

如果两个 agent 需要看到对方的代码改动，正确做法是：agent A 提交自己的分支，agent B 通过 `git fetch` 或 `git merge` 拉取。不要试图让 agent B 直接读取 agent A 的工作目录（worktree 路径），因为 B 的 agent 可能没有权限访问或路径不确定。

**原则三：状态文件追加写，不覆盖**

当多个 agent 都要更新 `TODO.md` 时，使用追加（append）而非覆盖（overwrite）。在 prompt 中明确告诉 agent："在 TODO.md 末尾追加一行，不要修改已有内容。"

---

## 六、常见协作模式

下面给出三种经过实践验证的多 agent 协作模式。每种模式给出适用场景和一个最小示例。

### 6.1 模式一：主管-工人（Orchestrator-Worker）

一个"主管 agent"负责拆解任务、分配给多个"工人 agent"、汇总结果。

**适用场景**：任务可以明确拆成互不依赖的子任务，且主管不需要亲自写代码。

**操作步骤：**

1. 启动 tmux，创建主管窗格 + N 个工人窗格
2. 主管窗格启动 agent，prompt 为：
   ```
   你是主管。任务：重构用户模块。请把任务拆成 3 个子任务，写入 agents/TODO.md，然后通知我开始。
   ```
3. 主管输出任务分配后，在每个工人窗格启动 agent，各自负责一个子任务
4. 工人完成后写入 `handoff/`，主管读取后汇总

**注意**：主管 agent 本身也可能在上下文增长后遗忘早期约定。实践中，每完成一个子任务就让主管重新读取一次 TODO.md 和 handoff/ 目录，比一次性记住所有信息更可靠。

### 6.2 模式二：对等互不干扰（Isolated Peers）

多个 agent 各自负责代码库的不同部分，从始至终不需要通信。

**适用场景**：大型代码库中多个模块同时开发，或一个 agent 写前端、一个写后端。

**操作步骤：**

1. 按第四节流程为每个 agent 创建独立 worktree
2. 每个 agent 的 prompt 中明确"你只负责 src/module-X/ 目录，不要修改其他目录"
3. 各自完成后合并分支

**优点**：无需通信协议，实现最简单。  
**缺点**：模块间接口变更时容易产生集成问题，建议最后留一个"集成 agent"做端到端测试。

### 6.3 模式三：评审循环（Review Loop）

一个 agent 写代码，另一个 agent 审查，循环直到通过。

**适用场景**：对代码质量要求高、单 agent 容易引入隐蔽错误的场景（安全相关、性能敏感路径）。

**操作步骤：**

1. Pane 0：agent-write，负责实现功能并提交
2. Pane 1：agent-review，prompt 为：
   ```
   你是代码审查员。每次 agent-write 提交后，运行 git diff main..feat-X 查看改动，检查：
   - 是否有未处理的错误
   - 是否有安全漏洞（SQL 注入、XSS）
   - 是否有明显性能问题
   
   如果有问题，把问题写入 agents/review-comments.md，格式：
   ## 第 N 轮审查
   - [file:line] 问题描述
     建议：xxx
   
   如果通过，写入：LGTM（Looks Good To Me）
   ```
3. 你在 Pane 0 读取审查意见，告诉 agent-write 修改，循环直到 LGTM
4. 合并到 main

**实际数据**：根据多个团队的实践记录，评审循环通常 2–4 轮收敛。超过 4 轮往往意味着任务拆分不够细，建议回到模式一重新拆分子任务。

---

## 七、取舍与踩坑

### 7.1 何时不该用多 agent

| 情况 | 原因 |
|------|------|
| 任务总时长 < 5 分钟 | 启动多个 agent 的 overhead（建 worktree、分窗格、写 prompt）超过并行收益 |
| 子任务之间有复杂依赖 | 通信成本高于串行执行，容易产生"改了又改"的循环 |
| 你不熟悉 tmux 基本操作 | 花在记住快捷键的时间超过开发时间 |

经验法则：只有当"任务可以拆成 ≥ 2 个互不依赖的子任务，且每个子任务预计 > 10 分钟"时，多 agent 才值得。

### 7.2 Token 成本

每个 agent 独立消耗自己的 token。三个并行 agent 的总 token 消耗通常是单 agent 的 2–3 倍（因为上下文无法共享，每个 agent 都要重复读取项目背景）。在按 token 计费的 API 场景下，先估算成本再决定是否拆分。

2026 年的模型价格（境内可查的公开信息）使得"单任务消耗 10 万–50 万 token"成为常见范围。如果三个子任务并行，准备 30 万–150 万 token 的预算。

### 7.3 Debug 难度

多 agent 出问题时，定位比单 agent 困难：

- agent A 改了接口签名，agent B 不知道，在合并时才发现冲突
- 两个 agent 使用了同一个临时文件路径，互相覆盖
- 某个 agent 在后台静默失败（如 API 超时），你直到合并时才察觉

**建议**：先从小规模开始（2 个 agent），积累通信约定后再扩展到 3–4 个。每次实验后记录"哪个 agent 出了什么问题、如何发现的"，形成自己的排查清单。

### 7.4 平台内置子代理 vs 手动编排

当前主流终端 agent（Claude Code、Codex CLI、Grok Build）都支持"子代理"（subagent）功能——你在主会话里下达指令，框架自动拉起子会话执行。

**何时用平台内置**：
- 子任务明确、单次执行
- 不需要人工中途干预
- 信任框架的上下文管理

**何时手动编排**：
- 你需要实时观察每个 agent 的中间过程
- 不同 agent 需要在不同目录（worktree）工作
- 你想用不同模型或不同配置（温度、工具集）运行不同 agent
- 你需要精确控制并行顺序（如"等 agent A 完成后再启动 agent C"）

手动编排的学习曲线更陡，但可控性更强。建议先用平台内置子代理跑通流程，遇到瓶颈再切换到手动 tmux + worktree 方案。
