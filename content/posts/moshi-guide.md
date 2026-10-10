---
title: "Moshi 与 moshi-hook 使用笔记"
date: 2026-10-10T16:29:27+08:00
draft: false
tags: ["工具链"]
description: "Moshi 是在手机上远程操作 AI 编码工具的终端应用，moshi-hook 是让终端认识这些工具的本机守护进程。这篇笔记记录安装、配对、装钩子、自检和排错的完整流程。"
---

## 1. 先说清楚两个名字

Moshi 是一个手机终端应用，支持 iOS、Android 和桌面端。它解决的问题是：AI 编码工具跑在自己的电脑上，人不在电脑前的时候，怎么继续盯着它、给它派活、在它弹出"是否批准"的时候做决定。

Moshi 连的是你自己机器上的真实终端，走 SSH、Mosh 或 Eternal Terminal。没有中转服务器，终端、仓库、文件、Agent 进程都留在你自己的机器上。它和普通 SSH 客户端的区别在于围绕 AI 工具做了一套原生界面：统一的收件箱、锁屏实时活动、Apple Watch 上直接点批准、Diff 查看、文件浏览、设备端语音输入。

`moshi-hook` 是装在**被连接那台机器上**的命令行程序，和 App 是两回事，可以只用 App 不装它。装了之后，Moshi 才认得出终端里跑的是哪个 AI 工具、它现在是闲着还是在等人批准。

本机装好之后，`moshi` 这个命令是指向 `moshi-hook` 的软链接，二者完全等价：

```bash
ls -l ~/.local/bin/moshi
# moshi -> moshi-hook
```

这篇笔记的版本依据是 `moshi-hook 0.4.7`。命令以 `moshi-hook --help` 的输出为准。

---

## 2. 两层各自负责什么

| | Moshi（手机端） | moshi-hook（机器端） |
| :--- | :--- | :--- |
| 形态 | App | 后台守护进程 + 命令行 |
| 连接对象 | 你的机器 | 本机的 AI 工具配置 |
| 提供的能力 | 终端、收件箱、锁屏活动、手表批准、语音、Diff、文件浏览、内嵌浏览器预览 | 给各 AI 工具写钩子、提供本地通信接口、维持到 Moshi 服务器的长连接、统计用量 |

moshi-hook 具体做三件事：

1. **往 AI 工具里写钩子**。在 Claude、Codex、opencode 等工具自己的配置目录里放脚本，工具状态一变就把事件报出来。
2. **提供本机 Unix socket**。钩子通过这个 socket 把事件送进守护进程。
3. **维持到 Moshi 服务器的 WebSocket**。把"需要人批准"这类事件送到手机，再把手机上的批准结果送回终端，完成一次往返。

要注意 moshi-hook **不接管任何 Agent 的运行**。它只是在旁边加了一层能看清 Agent 状态的外皮。终端、Diff、文件、预览这些流量仍然是手机和你的机器之间直接传的。

---

## 3. 安装

官方给的方式是一行脚本：

```bash
curl -fsSL https://getmoshi.app/install.sh | sh
```

支持 macOS、Linux 和 Windows 上的 WSL。也可以用 Homebrew，仓库地址是 `rjyo/homebrew-moshi`。装完之后把它注册成后台服务：

```bash
moshi-hook service install
```

在 Linux 上这会建一个 systemd 用户服务。查看状态：

```bash
moshi-hook service status
moshi-hook service uninstall
```

先确认服务跑起来了：

```bash
moshi-hook probe
```

会告诉你是不是装好了、进程在不在、本地网关通不通、版本号和本机编号。

---

## 4. 配对

App 和机器要互相认一下。在 Moshi App 的「设置 → 集成」里拿一个配对码，然后在机器上执行：

```bash
moshi-hook pair --token <配对码>
```

* `--name`：这台机器在 App 里显示的名字，默认用主机名。
* `--store`：密钥存哪儿。macOS 默认用系统钥匙串；没有钥匙串的环境（比如无界面 SSH 会话）用 `--store file`。

配对信息分两处存放：令牌和主机密钥放在密钥库里，主机编号单独放在旁边的 `host.json` 里，这样 SSH 会话也能读到。

查看配对状态：

```bash
moshi-hook status
moshi-hook status --json
```

输出包括配对状态、主机编号、显示名、用的服务器地址、密钥库类型、socket 路径和日志路径。本机显示的是已配对，Moshi Pro 未挂载，所以用量同步只在已配对设备之间进行。

解除配对用 `moshi-hook unpair`。

---

## 5. 给 AI 工具装钩子

配对好只是机器和 App 认识了，还得让各个 AI 工具会汇报状态：

```bash
moshi-hook install
```

默认给所有支持的工具都装。只想装一部分：

```bash
moshi-hook install --target claude,codex,opencode
```

当前版本支持的目标有 17 个：

```text
claude, codex, opencode, gemini, antigravity, cursor, kimi, qwen,
qoder, droid, copilot, amp, devin, grok, omp, pi, hermes
```

如果只想对某个项目生效、不想改全局配置，加 `--local`，钩子会写在当前项目里。

卸载用 `moshi-hook uninstall`，参数和 install 一样。

**一个容易踩的坑**：装钩子之前就已经开着的 Agent 会话，不会自动用上钩子，要重启那些会话才生效。

---

## 6. 自检：doctor

这是最该先跑的一条命令：

```bash
moshi-hook doctor
```

它逐项检查守护进程、本地网关、tmux/herdr/zellij、各个 Agent 的钩子、配对状态和设置，然后按"功能"汇报能不能用、缺什么。

按功能分的几类：

| 功能 | 依赖什么 |
| :--- | :--- |
| Agent 收件箱与提醒 | 各 Agent 的钩子 + 配对 |
| Chat View（把 Agent 界面渲染成对话） | 钩子，且只支持一部分 Agent |
| 工作区侧栏 | tmux 或 herdr |
| 会话选择器 | tmux 或 herdr |
| Diff 查看器 | Git |
| 浏览器与模拟器预览 | 本机 HTTP 服务探测 |
| 用量 | 各 Agent 的本地记录和账号接口 |

要机器可读的结果（比如给脚本或 App 用）：

```bash
moshi-hook doctor --json
```

它会列出每一项的状态、原因，以及具体该跑哪条修复命令。确认要修就加 `-y`，否则会先问：

```bash
moshi-hook doctor -y
```

看日志：

```bash
moshi-hook logs
moshi-hook logs -f          # 持续跟踪
moshi-hook logs -n 100      # 只看最后 100 行
```

---

## 7. 设置

```bash
moshi-hook set                          # 列出所有设置和当前值
moshi-hook set always-on-discovery      # 看某一项
moshi-hook set always-on-discovery off  # 改
```

可用的设置：

| 设置 | 默认 | 作用 |
| :--- | :--- | :--- |
| `always-on-discovery` | 开 | 没连手机时也继续找本机服务和模拟器 |
| `scan-ports` | `all` | 只探测哪些本机端口找 HTTP 服务；写 `3000,5173` 或 `3000,8000-8010`，`none` 关闭 |
| `usage-collection` | 开 | 后台轮询各 Agent 限额并同步到 Moshi；可写间隔，如 `5m`，`off` 关闭 |
| `git-background-fetch` | 关 | 后台 fetch 上游，让"落后几个提交"保持新鲜。开可能触发 SSH 或凭据弹窗 |
| `suppress-nested-agent-push` | 关 | 不推送由另一个 Agent 启动的 Agent（它们的批准请求也一起不推） |
| `suppress-push-while-unlocked` | 关 | 这台 Mac 解锁时静音推送，仅 macOS |

改布尔类设置后要重启守卫进程才生效，改扫描端口下次探测时就生效。

手动编辑 `~/.config/moshi/config.toml` 效果一样，文件里的注释和其他键会保留。安装脚本在检测到终端时会自动跑一次 `set --first-run`，让你一次性选默认项；Homebrew 安装不会自动跑，需要时自己执行：

```bash
moshi-hook set --first-run
```

---

## 8. 给手机端提供的几个查询

这几个命令主要是给 Moshi App 内部调用的，手动跑可以用来排查。

### context

回报当前终端的状态。不传参数时它自己判断环境：

```bash
moshi-hook context
```

判断顺序是：有 `$TMUX_PANE` 按 tmux 处理，有 `$HERDR_ENV=1` 按 herdr 处理，有 `$ZELLIJ` 按 zellij 处理，都不是就按普通 shell 处理。返回的内容包括所在会话、窗格编号、工作目录、Git 状态，以及识别出的 Agent 名字和状态。

针对远端会话查询时，用这三个参数指定目标，App 靠它们找到对应会话：

```bash
moshi-hook context --ssh-connection "$SSH_CONNECTION"
moshi-hook context --mosh-port 60001
moshi-hook context --et --et-client-id <id>
```

### cwd-list

扫本机 Agent 留下的记录（Claude 的会话记录、Codex 的 rollout 文件、Cursor 的项目目录），列出去重后的最近工作目录，按新鲜度排序：

```bash
moshi-hook cwd-list
moshi-hook cwd-list --limit 20 --json
```

App 连接时用它做"最近目录"列表，让你一键跳回某个项目。

### servers

探测本机的 TCP 监听端口，找出 HTTP 服务，供 App 里"打开本机 dev server"用。带 `kill` 子命令可以杀掉发现的服务。

### usage

汇总各 Agent 账号的限额快照。数据来自两处：各 Agent 自己的本地文件，以及只读的账号额度接口（Codex、Claude、Kimi Code、Grok Build）。默认只打印，加 `--sync` 才上传到 Moshi 服务器：

```bash
moshi-hook usage
moshi-hook usage --sync
```

---

## 9. 本机就能用的两个功能

除了配合手机，moshi-hook 在本机也有两个能直接用的功能。

**本地 Web 客户端**。不带参数直接跑：

```bash
moshi-hook
```

会在 `127.0.0.1:24544` 起一个网页客户端，直到按 Ctrl-C。用 `--listen` 改地址，想对外开放才需要 `0.0.0.0`；`--no-open` 只打印地址不自动开浏览器。

**Git Diff 查看器**：

```bash
moshi-hook diff ~/project
moshi-hook diff --no-open        # 只打印本地地址
```

在浏览器里看某个目录的 Git 改动，直接读 Git 状态，不经过 Moshi 服务器。端口默认 24543，写 0 会自动挑一个空闲端口。

另外 `moshi-hook .` 可以为当前目录开一个 tmux 会话或接回已有的，等价于把路径交给 tmux 处理。

---

## 10. SSH 与 Mosh 接入

如果想让 App 通过 SSH 或 Mosh 连进来（而不是只走长连接那一套），要单独准备：

```bash
moshi-hook host setup
```

可选项：`--name` 显示名，`--host` 对外宣告的地址或 IP，`--port` SSH 端口（默认 22），`--user` 登录用户（默认当前用户），`--json` 输出机器可读结果，`--force` 在前提检查报警告时继续。

它会在 `authorized_keys` 里装一个专用密钥。查看和管理：

```bash
moshi-hook host list                 # 看已装的密钥
moshi-hook host revoke <编号>          # 从 authorized_keys 移除某个
moshi-hook host enable-ssh           # 打开系统的 SSH 服务
```

`enable-ssh` 在 macOS 上会跑 `sudo systemsetup -setremotelogin on`，需要输密码；如果系统拦住了（新版 macOS 要求调用方终端有完全磁盘访问权限），它会打开共享设置面板让你手动打开。

---

## 11. 和 herdr 一起用

如果你用 herdr 管终端，这两层是配合关系：herdr 负责把本机的窗口归置整齐并识别 Agent 状态，moshi-hook 负责把这些状态送到手机上。

moshi-hook 认 herdr 靠的是环境变量。在 herdr 管理的窗格里会注入 `HERDR_ENV=1`、`HERDR_WORKSPACE_ID`、`HERDR_TAB_ID`、`HERDR_PANE_ID`。`moshi-hook context` 检测到 `HERDR_ENV=1` 就按 herdr 处理，回报工作区编号和标签页编号，App 侧栏据此显示各个 herdr 标签页的 Agent 状态，并支持直接跳转过去。

这正好接上 herdr 那篇笔记里第 12 节的内容：那组环境变量除了给 AI 助手用，也是 moshi-hook 识别 herdr 会话的依据。

---

## 12. 实际会碰到的问题

这一节记一下本机使用中真实出现的情况，按出现频率排。

### 钩子过期

这是最容易遇到、影响也最大的。表现是 `moshi-hook doctor` 里收件箱那一项失败，提示某个 Agent 的钩子过期。日志里会反复出现 `agent hooks missing or stale; rerun install`。

钩子过期的后果是：手机收不到批准请求、没有推送、没有锁屏活动，moshi-hook 最有价值的部分等于没开。修的办法是对每个过期的 Agent 跑一次：

```bash
moshi-hook install --target <名字>
```

`moshi-hook doctor --json` 会把该跑的命令直接列出来。

### WebSocket 频繁掉线

日志里 `ws bridge dropped` 的次数明显多于 `ws bridge connected`，每次掉线后退避 1 秒重连。手机端表现是事件延迟或短暂收不到。这类重连本身是正常设计，次数偏多时先查网络，尤其是机器在 NAT 后面或者用了代理的情况。

### 某几个 Agent 的用量拉不到

日志里形如 `usage fetcher: repeated poll failures; cached usage is stale` 的警告，会带上 Agent 名。说明这个账号的限额读不到了，App 上显示的用量是旧的缓存。可以单独确认：

```bash
moshi-hook usage
```

如果本地文件的快照也读不到，就只能等网络或账号侧恢复。

### 排查顺序

出问题时按这个顺序查，基本能定位：

1. `moshi-hook probe` —— 装没装、进程在不在、网关通不通
2. `moshi-hook service status` —— 后台服务是不是活着
3. `moshi-hook doctor` —— 逐项看哪一层断了
4. `moshi-hook logs -n 200` —— 看具体报了什么
5. `moshi-hook status` —— 配对还在不在

---

## 13. 小结

Moshi 和 moshi-hook 是明确分工的两层：App 负责手机端的终端和交互，moshi-hook 负责让本机的 AI 工具会汇报状态。装 moshi-hook 不改变 Agent 的任何运行方式，只是让它们的状态对外可见。

真正要留意的有两件事。第一，装完钩子一定要跑一次 `moshi-hook doctor` 确认通过，并且重启已开着的 Agent 会话，否则钩子不生效。第二，用量同步和部分功能（`Chat View`、`unlimited hosts` 等）需要 Moshi Pro，免费额度之外的功能要先确认自己的授权状态。

对已经用 herdr 的人来说，这套东西补上的是"离开电脑之后还能不能管住这些 Agent"这一环。herdr 解决本机的窗口秩序，moshi-hook 和 Moshi 解决远程可见和远程批准。
