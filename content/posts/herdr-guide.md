---
title: "herdr 使用教学笔记"
date: 2026-10-10T16:19:15+08:00
draft: false
tags: ["工具链"]
description: "herdr 是终端里管理多个 AI 编码工具的软件。这篇笔记记录它的分层结构、常用命令、快捷键和配置方法。"
---

## 1. herdr 是什么

herdr 是一个在终端里运行的界面管理工具，用一句话概括：它把终端分成工作区、标签页和窗格，用来同时运行多个命令行程序，并且能认出跑在窗格里的是哪个 AI 编码工具。

平时我们在终端里跑 Claude Code、Codex、opencode 这类工具，窗口一多就乱了：不知道哪个在哪，哪个还在等确认，哪个已经做完。herdr 解决的问题就是这个。它把这些窗口归置整齐，还能看出每个 AI 工具现在是空闲、在干活，还是弹出了需要人确认的对话框。

herdr 分客户端和服务端。服务端在后台常驻，负责记住所有窗口；客户端只是连上去看。所以把终端关了、甚至把 SSH 断开，里面的任务照样在跑，下次连回来还能接着看。这一点是和普通终端窗口最大的区别。

这篇文章记录的是本机 herdr `0.9.3` 的用法。它的命令比较多，实际记的时候以 `herdr --help` 的输出为准，那个最准。

---

## 2. 四层结构：会话、工作区、标签页、窗格

理解 herdr 的关键是记住它分四层，从大到小排：

| 层级 | 作用 | 编号样子 |
| :--- | :--- | :--- |
| 会话（session） | 一套独立运行的环境，有自己的服务端进程 | 用名字，比如 `default` |
| 工作区（workspace） | 一组标签页，通常对应一个项目目录 | `w1` |
| 标签页（tab） | 一组窗格，用来再分一层 | `w1:t1` |
| 窗格（pane） | 真正跑程序的最小格子 | `w1:p1` |

编号是 herdr 自动给的，不用去记，用命令查出来就行。要注意编号是"一次性"的：窗格关掉之后，那个编号不会被下一个新窗格复用。窗格被挪到别的工作区，编号也会变。所以命令行里传编号时，要用最近查出来的那个。

会话默认有一个叫 `default` 的。想另起一套，用 `herdr --session <名字>`。

---

## 3. 装好之后先查状态

```bash
herdr status
```

这条命令会同时看客户端和后台服务端的情况，包括版本号、能不能对上话、用的是哪个通信文件。加 `server` 或 `client` 可以只看一边：

```bash
herdr status server
herdr status client
```

正常情况下输出类似这样：

```text
client:
  version: 0.9.3
  channel: stable
  server:
  status: running
  socket: /home/xfz/.config/herdr/herdr.sock
```

版本号两边的要一致。如果提示对不上，一般是升级了一半，跑一下 `herdr update` 再重试。

---

## 4. 启动和退出

直接敲 `herdr` 就能打开界面。如果之前有会话在跑，就是连回去；没有就新建。

日常常用的是带名字的会话：

```bash
herdr --session work
```

这样打开的是一个叫 `work` 的会话，任务都在里面跑。以后随时可以：

```bash
herdr session list          # 看有哪些会话
herdr session attach work   # 连回某个会话
herdr session stop work     # 停掉某个会话
herdr session delete work   # 删掉已停掉的会话
```

退出界面用 `Ctrl+b` 然后按 `q`（具体快捷键见第 8 节）。这只是"断开"，不是"关掉"，里面的程序照跑。

要真正停掉后台服务：

```bash
herdr server stop
```

改完配置文件不想重启整个会话，可以让服务端重新读一遍：

```bash
herdr server reload-config
```

**这里有个坑**：想看用法的时候不要直接敲 `herdr`，这一敲就把界面打开了。要看用法就敲 `herdr --help`。

---

## 5. 窗格：最基本的操作

命令行操作 herdr 时，打交道最多的就是窗格。所有命令都是操作"已经在跑的界面"，不会自己跳出来一个新窗口。

### 5.1 先看看现状

```bash
herdr pane list              # 列出所有窗格
herdr pane current --current # 看自己所在的窗格
herdr pane layout --current   # 看当前窗格是怎么摆的
```

输出的格式是 JSON，直接读里面的字段就行，比如 `pane_id`、`cwd`、`agent_status`。

### 5.2 打开一个窗格

```bash
herdr pane split --current --direction right --cwd "$PWD" --no-focus
```

几个参数分开说：

* `--current`：在当前窗格旁边开，而不是在光标停的那个窗格旁边。命令行里自己跑的时候经常不在 herdr 界面里，加上它最稳。
* `--direction right`：往右开。当前窗格比较宽就往右，比较窄或者比较高就换 `down`。同一个方向连着开会越开越窄，不好用。
* `--cwd "$PWD"`：新窗格用现在这个目录。新窗格默认会跟着来源窗格的目录，但显式写出来更保险。
* `--no-focus`：光标别跳到新窗格，还留在原地。这个在脚本里很有用，省得打断人正在看的界面。

命令返回的 JSON 里，`.result.pane.pane_id` 就是新窗格的编号，后面要用。

### 5.3 在新窗格里跑命令

```bash
herdr pane run w1:p2 "just test"
```

这条命令会把命令文字和回车一起发过去，比手动 `send-text` 再 `send-keys enter` 可靠。

跑完之后想确认结果，有两种办法。等某行字出现：

```bash
herdr pane wait-output w1:p2 --match "test result" --timeout 120000
```

`--match` 是找一段原文，`--regex` 是写正则。不加 `--timeout` 会一直等。

或者直接把输出读出来：

```bash
herdr pane read w1:p2 --source recent-unwrapped --lines 120
```

读输出有四种来源，按需要选：

| 来源 | 是什么 |
| :--- | :--- |
| `visible` | 屏幕上现在能看到的部分 |
| `recent` | 最近一段输出，包含折行 |
| `recent-unwrapped` | 最近一段输出，把折行接回去。看日志和对话记录用这个 |
| `detection` | 用来判断 AI 工具状态的那份纯文本 |

### 5.4 其他窗格命令

* `herdr pane focus <编号>`：把光标移到某个窗格
* `herdr pane zoom <编号>`：把某个窗格放大到占满，再执行一次还原
* `herdr pane resize ...`：调整分割比例
* `herdr pane rename <编号> <名字>`：给窗格改个名
* `herdr pane send-text <编号> <文字>`：只发文字，不发回车
* `herdr pane send-keys <编号> <按键>`：发按键，比如 `esc`、`ctrl+c`
* `herdr pane swap`、`herdr pane move`：交换位置、挪到别处
* `herdr pane close <编号>`：关掉窗格
* `herdr pane process-info <编号>`：看窗格里跑的是什么进程

---

## 6. 让 herdr 认出 AI 工具

herdr 的特别之处是能认出窗格里跑的是哪个 AI 编码工具，并判断它现在的状态。要这个功能，得先装对应的"集成"。

### 6.1 装集成

```bash
herdr integration status
```

这条命令会列出所有支持的 AI 工具，以及本机装了没有、版本新旧。输出像这样：

```text
claude: current (v10) (/home/xfz/.claude/hooks/herdr-agent-state.sh)
droid: not installed (/home/xfz/.factory/hooks/herdr-agent-state.sh)
```

`current` 表示已装且是最新的，`not installed` 表示没装，`outdated` 表示有新版。装和卸用：

```bash
herdr integration install claude
herdr integration uninstall claude
```

集成干的事，是在各个 AI 工具自己的配置目录里加一个钩子脚本，工具状态一变就告诉 herdr。所以装完之后，herdr 显示的状态才是准的。

### 6.2 状态的五种含义

| 状态 | 意思 |
| :--- | :--- |
| `idle` | 空着，可以给它派活 |
| `working` | 正在干活 |
| `blocked` | 弹出了要人批准的界面，在等人 |
| `done` | 活干完了，可以派下一个（和 `idle` 的区别是"看没看到"） |
| `unknown` | 里面是有个 AI 工具，但认不出具体在干什么 |

`idle` 和 `done` 都能派活。区别在于 herdr 记不记得你看过这条完成消息：命令行里读过就算看到，界面里光标停过也算。不同端看到的可能不一样，这是正常的。

### 6.3 在窗格里启动一个 AI 工具

```bash
herdr agent start reviewer --kind codex --pane w1:p2
```

* `reviewer`：给这个 AI 工具起的名字，方便后面用名字而不是编号来指它。名字只能用小写字母开头，后面跟小写字母、数字、下划线和连字符，最长 32 个字符，而且不能重名。
* `--kind codex`：工具类型。执行 `herdr agent start --help` 能看到完整清单，`codex`、`claude`、`opencode`、`gemini`、`copilot`、`kimi`、`grok` 等都在里面。
* `--pane w1:p2`：在哪个窗格里启动。这个窗格必须停在一个空白的命令行提示符上，里面不能有正在跑的前台程序、编辑器或者别的 AI 工具。

这条命令不会自己开新窗格。要先 `pane split` 开一个，再把 AI 工具放进去。命令成功返回，说明 herdr 已经在这个窗格里认出目标工具了，而且它准备好看输入。

想给 AI 工具带参数，写在 `--` 后面：

```bash
herdr agent start reviewer --kind codex --pane w1:p2 -- --model gpt-5
```

如果启动过程中它弹出了待确认界面，命令会返回 `agent_not_ready`，但名字已经占上了，可以先用 `agent read` 和 `agent send-keys` 去处理。

### 6.4 派活和等结果

```bash
herdr agent prompt reviewer "检查当前改动，只报告需要处理的问题" --wait --timeout 120000
```

命令行里的目标可以写名字，也可以写它所在的窗格编号。写名字更稳，因为窗格挪动之后编号会变。

关于 `--wait`：

* 它会等到出现三种状态之一：`idle`、`done`、`blocked`。
* 为了确认活真的发出去了，派活之后 5 秒内必须看到 `working` 或 `blocked`。要是没动静，返回 `agent_prompt_stalled`，意思是可能没送进去。
* `--timeout` 是总时限，包括派活本身花的时间。
* 它只认"状态稳了"，不认"这一轮结束了"。如果它本来就在干活，那当前这轮做完也可能让等待结束。

只想等某个特定状态，用 `--until`：

```bash
herdr agent wait reviewer --until blocked --timeout 120000
```

不加 `--until` 的话，`agent wait` 和 `agent prompt --wait` 的默认行为一样。

### 6.5 看它在干什么、发按键

```bash
herdr agent read reviewer --source recent-unwrapped --lines 120
herdr agent get reviewer
```

`agent get` 看元信息（状态、目录、编号），`agent read` 看屏幕内容。

给它的界面发按键，逻辑键名而不是字符：

```bash
herdr agent send-keys reviewer esc       # Esc
herdr agent send-keys reviewer ctrl+c    # 中断
```

其它还有 `herdr agent attach`（直接接进它的终端）、`herdr agent rename`、`herdr agent focus`、`herdr agent list`。

状态判断拿不准的时候，用：

```bash
herdr agent explain --verbose
```

它会说明为什么判定成这个状态。排查"herdr 明明忙着我却显示空闲"这类问题，先跑这个。

### 6.6 遇到 blocked 怎么办

`blocked` 说明它停在要批准的界面。这时候 `agent prompt` 会直接拒绝，不发送任何东西，返回 `agent_blocked`，免得你把新任务混进旧对话框里。

正确做法是先看它要批什么：

```bash
herdr agent get reviewer
herdr agent read reviewer --source visible
```

确认是给谁批、批什么，再决定。这一步别省，也别让别的程序替你按同意。

---

## 7. Git worktree：一个活儿一个目录

herdr 能直接对着 Git worktree 开工作区。几个 AI 工具同时改一个仓库时，各占一个 worktree，互不干扰。

```bash
herdr worktree create --cwd ~/project --branch fix-login --base main --label "修登录"
```

* `--cwd`：仓库位置
* `--branch`：新分支名
* `--base`：从哪个提交或分支切出来
* `--path`：worktree 放哪，不填会自己选位置
* `--label`：侧栏显示的名字
* `--focus` / `--no-focus`：要不要把光标切过去

相关命令：

```bash
herdr worktree list                          # 看有哪些 worktree
herdr worktree open --path <路径>             # 打开已有的 worktree
herdr worktree remove --workspace <编号>       # 移除一个 worktree
```

注意 `worktree create` 和 `workspace create` 一样，不带参数也会真的创建，试探用法前先想清楚。

---

## 8. 快捷键

herdr 的快捷键和 tmux 一样是"前缀键"模式。默认前缀是 `Ctrl+b`，先按它，松手，再按功能键。

最常用的几个：

| 快捷键 | 功能 |
| :--- | :--- |
| `Ctrl+b` `d` | 断开会话（程序继续跑） |
| `Ctrl+b` `c` | 新建标签页 |
| `Ctrl+b` `v` | 左右分割窗格 |
| `Ctrl+b` `-` | 上下分割窗格 |
| `Ctrl+b` `x` | 关掉当前窗格 |
| `Ctrl+b` `z` | 当前窗格放大 / 还原 |
| `Ctrl+b` `h` `j` `k` `l` | 在窗格间移动光标 |
| `Ctrl+b` `Tab` | 挨个切换窗格 |
| `Ctrl+b` `1`..`9` | 按序号切标签页 |
| `Ctrl+b` `n` / `p` | 下一个 / 上一个标签页 |
| `Ctrl+b` `w` | 打开工作区选择器 |
| `Ctrl+b` `?` | 帮助 |

改快捷键在配置文件里，见下一节。

---

## 9. 配置文件

配置文件默认在：

```text
~/.config/herdr/config.toml
```

用环境变量 `HERDR_CONFIG_PATH` 可以指到别处。

想看全部可选项和默认值，运行：

```bash
herdr --default-config
```

它会打印一份带注释的完整配置，照着改就行。日志也在同一目录下：`herdr.log`、`herdr-client.log`、`herdr-server.log`，出问题时先看这里。

几个常用设置：

```toml
[theme]
name = "catppuccin"        # 也有 terminal、tokyo-night、dracula、nord 等
auto_switch = true         # 跟随终端深浅色自动切换

[terminal]
default_shell = "/bin/zsh" # 新窗格用什么 shell
new_cwd = "follow"         # 新窗格的目录：follow / home / current / 固定路径

[update]
channel = "stable"         # 更新通道：stable 或 preview
version_check = true       # 后台检查新版本

[keys]
prefix = "ctrl+b"          # 前缀键
```

改完先检查有没有写错：

```bash
herdr config check
```

显示 `config: ok` 就行了，然后让运行中的服务端重新加载：

```bash
herdr server reload-config
```

只想把改乱的快捷键恢复默认：

```bash
herdr config reset-keys
```

它会先备份原文件，再清掉自定义按键，所以不会丢别的配置。

---

## 10. 远程机器

herdr 可以通过 SSH 管另一台机器上的界面。

先加一台：

```bash
herdr machine add user@server.example.com --label 集群
```

它会自动在远端把 herdr 服务准备好并存成一个档案。之后的命令都加上 `--machine` 前缀，指哪台就操作哪台：

```bash
herdr --machine 集群 agent list
herdr --machine 集群 pane list
herdr --machine 集群 agent prompt worker "汇报进度" --wait --timeout 120000
```

几条要注意的：

* `--machine` 后面写的是档案编号或标签，不能直接写 SSH 地址。
* `--machine` 不要和 `--session`、`--remote` 一起用。
* 远端的编号要在远端查，本地的编号拿到远端不好使。
* 连接失败不代表命令没执行，重试之前先去远端看一眼状态。
* 加机器时如果远端服务端版本不兼容，它会问要不要停掉旧的。默认是"否"，别随手按同意。

管理档案：

```bash
herdr machine list                          # 看存了哪些
herdr machine status 集群                    # 测某台通不通（不会要密码）
herdr machine reconnect 集群                  # 重新认证，要人在终端里输密码
herdr machine rename 集群 新名字
herdr machine disable 集群                   # 暂时停用
herdr machine remove 集群                    # 删档案
```

`machine reconnect` 需要本人在终端里完成 SSH 认证，不要代劳。

---

## 11. 一些零碎但常用的命令

```bash
herdr workspace list                  # 看所有工作区
herdr workspace create --cwd ~/app    # 新建工作区（会真的建）
herdr workspace rename <编号> <名字>
herdr workspace close <编号>

herdr tab list --workspace w1         # 看某个工作区的标签页
herdr tab create --workspace w1 --cwd ~/app
herdr tab focus <编号>

herdr notification show "任务完成" --body "tests passed" --position top-right
```

排查和查询：

```bash
herdr api snapshot          # 导出当前界面的完整状态
herdr api schema            # 看通信接口的定义，能写成脚本
herdr agent explain -v      # 解释状态判断
herdr config check          # 检查配置
herdr status                # 检查版本
```

更新自己：

```bash
herdr update
herdr channel set stable     # 或 preview
```

补全脚本：

```bash
herdr completion zsh
```

直接连到远端界面（不用先存档案）：

```bash
herdr --remote user@server
```

---

## 12. 给 AI 助手用的入口

herdr 给 AI 编程助手留了一个专门的接入方式。它会在管理的每个窗格里注入几个环境变量：

```bash
echo $HERDR_ENV            # 是不是在 herdr 管理的窗格里
echo $HERDR_WORKSPACE_ID   # 所在工作区编号
echo $HERDR_TAB_ID         # 所在标签页编号
echo $HERDR_PANE_ID        # 所在窗格编号
```

如果你在用的 AI 助手支持 herdr，它会读取内置的技能文件来学命令。手动查看这份文件：

```bash
herdr --skill
```

文件里有两条重要约束，值得单独记住：

第一，任何控制命令之前，先确认自己是不是真的跑在 herdr 的窗格里：

```bash
test "$HERDR_ENV" = "1"
```

不是的话就别去动 herdr 的界面，容易和用户正在做的事打架。

第二，看用法用 `herdr --help`，不要直接敲 `herdr`，也不要不带参数试探会改状态的命令（比如 `herdr workspace create`）。命令行里返回的基本都是 JSON，编号和状态从返回里读，不要靠猜。

---

## 13. 一份上手流程

最后把常用路径串一遍：

1. 装完跑 `herdr integration status`，把在用的 AI 工具集成装上。
2. 敲 `herdr` 进界面，`Ctrl+b` `d` 断开，任务继续跑。
3. 命令行里想给别的窗格派活：`pane list` 找编号 → `pane split` 开新格 → `agent start` 放工具 → `agent prompt` 派活 → `agent read` 看结果。
4. 只想跑个命令不想开 AI：`pane split` → `pane run` → `pane wait-output`。
5. 状态判断不对劲：`agent explain --verbose`。
6. 改完配置：`config check` → `server reload-config`。

herdr 的命令确实多，但常用的就是"看列表、开窗格、派活、读结果"这四步。剩下的用到再查 `herdr <命令组> --help` 就够了。
