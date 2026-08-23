# 第 2 课：Git + Coding Agent——为什么企业级 Agent 必须学会“隔离修改”

上一课建立了 `LLM → Tool Call → Shell → stdout/stderr → 再决策` 的 Agent Loop。今天加入第二个非常关键的基础设施：entity["software","Git","distributed version control system"]。

目标不是背 Git 命令，而是理解一个企业问题：**如果同时让 3 个 Agent 修改同一个项目，怎样防止它们互相覆盖代码？**

答案之一就是：

```text
Git branch + worktree + diff + test + review
```

这也是现代 Coding Agent 越来越重要的一种执行模式。近期 entity["software","OpenAI Codex","coding agent"] 和 entity["software","OpenCode","AI coding agent"] 的公开 issue 中，都能看到大量围绕 worktree、工作目录、diff 捕获和 Agent session 隔离的真实工程问题。citeturn0search2turn0search3

## 先建立 Git 的正确心智模型

假设项目当前是：

```text
sample-app/
├── src/
│   └── Program.cs
└── .git/
```

`.git` 可以暂时理解为项目的“版本数据库”。

你修改：

```text
Program.cs
```

但还没有 commit，此时处于：

```text
HEAD
 │
 ▼
上一次 commit
 │
 └──── working tree
          │
          └── Program.cs 已修改
```

因此：

```bash
git status
```

回答的是：

> 当前工作目录与 Git 记录的状态有什么不同？

而：

```bash
git diff
```

回答：

> 到底改了哪些内容？

比如 Agent 把：

```csharp
int timeout = 10;
```

改成：

```csharp
int timeout = 30;
```

`git diff` 会近似显示：

```diff
- int timeout = 10;
+ int timeout = 30;
```

这对 Agent 特别重要。

因为模型不能仅凭：

> “我刚才应该只改了 timeout。”

来判断自己做对了什么。

企业系统更可信的做法是：

```text
Agent的描述
      ↓
不完全可信

Git实际diff
      ↓
事实依据
```

这已经引出第一个 Agent 工程原则：

> **不要相信 Agent 声称修改了什么，要检查实际状态。**

---

## 为什么 Codex 会频繁调用 Git

现在看一个 2026 年非常有意思的真实案例。

Codex Desktop 在 Windows 上曾出现文件修改失败问题，issue 中发现后台存在类似：

```bash
git diff --no-ext-diff --no-textconv \
  --color=never \
  --find-renames \
  --numstat -z
```

这样的 Git diff 读取操作。citeturn0search6

今天不用理解所有参数。

只注意：

```text
git diff
```

Codex 并不是只在最后告诉你：

> “我修改了三个文件。”

它的运行环境需要持续掌握 repository 的真实变化。

于是 Coding Agent 实际架构逐渐变成：

```text
                 LLM
                  │
            Agent Runtime
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
     Shell      Files       Git
       │          │          │
 dotnet test    patch      status
 npm test       read       diff
 rg             write      branch
                           worktree
```

Git 实际上已经成为 Agent Runtime 的一个重要“状态传感器”。

---

# 为什么 branch 还不够

假设现在同时有两个任务：

```text
Agent A：
修复 HTTP timeout

Agent B：
重构日志模块
```

最简单想法：

```text
Agent A → branch feature/http-timeout

Agent B → branch feature/logging
```

但是存在一个问题：

普通 Git checkout 在一个目录里一次只能 checkout 一个 branch。

于是：

```text
sample-app/
    ↓
当前只能是一个 branch
```

如果 A 正工作：

```bash
git switch feature/http-timeout
```

B 又运行：

```bash
git switch feature/logging
```

A 的执行环境就被改变了。

对于并行 Agent，这很危险。

---

# Worktree 出场

Git 提供：

```bash
git worktree
```

它允许：

```text
sample-app/
│
├── main workspace
│
│   branch: main
│
├── ../sample-app-timeout
│
│   branch: agent/http-timeout
│
└── ../sample-app-logging
    branch: agent/logging
```

现在：

```text
Agent A
↓
sample-app-timeout/

Agent B
↓
sample-app-logging/
```

两者：

- 文件目录不同；
- branch 不同；
- working tree 不同；
- 但共享 Git repository 历史。

这就是 **Agent workspace isolation**。

---

# 真实命令

假设当前：

```bash
git branch
```

输出：

```text
* main
```

创建 Agent 专用工作区：

```bash
git worktree add ../sample-app-agent -b agent/test
```

拆开看：

```text
git
│
├── worktree
│     Git worktree 功能
│
├── add
│     创建一个工作目录
│
├── ../sample-app-agent
│     新目录
│
├── -b
│     同时创建 branch
│
└── agent/test
      branch 名称
```

然后：

```bash
git worktree list
```

可能得到：

```text
/Users/me/sample-app         abc123 [main]
/Users/me/sample-app-agent   abc123 [agent/test]
```

现在两个目录初始代码完全一样。

但是修改互不影响。

---

# 为什么这非常适合 Coding Agent

假设企业服务器同时收到：

```text
Task #101
修复登录问题

Task #102
增加导出Excel

Task #103
修复数据库连接
```

可以变成：

```text
                  Git Repository
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
      Worktree A    Worktree B    Worktree C
          │             │             │
       Agent A        Agent B       Agent C
          │             │             │
        Test           Test          Test
          │             │             │
        Diff           Diff          Diff
          │             │             │
          └─────────────┼─────────────┘
                        ↓
                      Review
                        ↓
                       Merge
```

注意：

**这已经非常接近一个 Mini Agent Platform 的 workspace manager。**

后面我们自己实现 Agent 时，也会真正做这一层。

---

# OpenCode 的真实案例

OpenCode 社区已经有专门围绕 Agent worktree 工作流的插件：

```text
Agent
 ↓
worktree_create()
 ↓
创建 Git worktree
 ↓
启动新 terminal
 ↓
在隔离目录运行 OpenCode
 ↓
Agent 完成工作
 ↓
commit
 ↓
删除 worktree
```

插件甚至支持创建 worktree 后执行：

```bash
pnpm install
```

或者：

```bash
docker compose up -d
```

删除之前：

```bash
docker compose down
```

这已经把今天的 Git 和后面要学的 **Docker 生命周期**连接起来了。citeturn0search0

注意：这个 worktree 插件是社区项目，不是 OpenCode 官方核心组件，因此学习时要区分“OpenCode 本身”和“生态插件”。

---

# Worktree 本身也会产生 Agent Bug

这一点特别值得企业工程师理解。

worktree 不是简单复制目录。

普通 repository：

```text
project/
└── .git/
```

但 linked worktree 中：

```text
project-feature/
└── .git
```

这里 `.git` 甚至可能**不是目录，而是一个文本文件**，指向：

```text
project/.git/worktrees/project-feature
```

这已经让不少 Coding Agent 出过问题。

2026 年 OpenCode 的一个公开 issue 就展示了这种情况：代码错误地按每个 worktree 的 `.git` 路径寻找项目缓存，导致同一个 repository 的不同 worktree 被识别成不同 project ID。问题定位到了 `src/project/project.ts` 的目录解析逻辑；建议修复方式之一就是正确使用：

```bash
git rev-parse --git-common-dir
```

来找到共享 Git 目录。citeturn0search3

这就是非常好的“源码 + Git + Agent 调试”案例。

你以后看到：

```text
Agent在主仓库正常

到了worktree：
session消失
项目识别错误
branch错误
```

第一反应应该增加：

```text
.git到底是目录还是pointer？
git-common-dir是什么？
cwd在哪里？
```

而不是马上认为：

> 模型出问题了。

---

# Codex 也有类似真实问题

2026 年 7 月 Codex App 有一个公开问题：

> 从任务创建 worktree fork 时，可能基于过时的默认 branch commit 创建新工作区，再把原任务的未提交 diff 应用过去。

结果就可能发生：

```text
原始任务
main @ commit C
     │
     ├── uncommitted diff
     │
     ↓
创建 worktree
     │
但基于旧 commit A
     │
     ↓
再应用 C 上产生的 diff
     │
     ↓
冲突 / 错误代码
```

这不是 LLM reasoning 问题。

这是：

```text
Git base commit
+
worktree provisioning
+
patch application
```

的问题。citeturn0search2

这类问题就是以后 Agent 平台工程师需要定位的。

---

# 一个必须掌握的词：CWD

以后会经常看到：

```text
cwd
```

就是：

**Current Working Directory**

当前工作目录。

例如：

```text
CWD =
C:\Projects\sample-app-agent
```

Agent 执行：

```bash
git status
```

实际含义是：

```text
在 C:\Projects\sample-app-agent
运行 git status
```

如果 Runtime 错误地变成：

```text
CWD =
C:\Projects\sample-app
```

即使执行完全相同：

```bash
git status
```

得到的 branch、diff、文件都可能不同。

所以以后调 Agent，一定要问：

```text
模型生成了什么命令？
        ↓
Runtime在哪个cwd执行？
        ↓
使用哪个shell？
        ↓
使用什么env？
        ↓
返回什么exit code？
```

这是非常重要的四连问。

---

# Windows / PowerShell 实验

今天实验大约 10 分钟。

找一个不重要的 Git 测试项目。

先执行：

```powershell
git status
git branch --show-current
git rev-parse --show-toplevel
```

分别观察：

```text
repository状态
当前branch
repository根目录
```

然后：

```powershell
git worktree list
```

创建实验 worktree：

```powershell
git worktree add ../agent-lab -b agent/lab
```

进入：

```powershell
cd ../agent-lab
```

执行：

```powershell
git branch --show-current
git rev-parse --show-toplevel
git status
```

应该看到 branch：

```text
agent/lab
```

然后创建：

```powershell
"hello agent" | Out-File agent-test.txt
```

再：

```powershell
git status
git diff
```

这里会出现一个有意思的现象：

`git status` 能看到：

```text
agent-test.txt
```

但普通：

```bash
git diff
```

可能看不到这个文件内容。

为什么？

因为它还是：

```text
untracked file
```

这就是下一步 Git 知识：

```text
untracked
working tree
index/staging
HEAD
```

---

# 一个非常重要的 Agent 安全规则

以后自己的 Agent 不应该简单实现：

```python
run("git add .")
run("git commit -m 'done'")
```

因为 `git add .` 可能把：

```text
.env
API Key
临时数据库
测试数据
大型模型文件
用户隐私文件
```

一起加入 repository。

企业 Agent 更合理的是：

```text
Agent修改
   ↓
git status
   ↓
确定changed files
   ↓
安全规则检查
   ↓
git diff
   ↓
test
   ↓
人工/策略审批
   ↓
选择性stage
   ↓
commit
```

以后讲 Secret、供应链安全和 CI/CD 时会再次回来。

---

# 今天第一次接触 `.gitignore`

假设项目：

```text
.env
node_modules/
bin/
obj/
```

通常不应该提交。

于是 `.gitignore`：

```gitignore
.env
node_modules/
bin/
obj/
```

Coding Agent 必须尊重这些规则。

但请记住：

> `.gitignore` 是 Git 的版本控制规则，不是安全边界。

Agent 仍然可能通过：

```bash
cat .env
```

读取 `.env`。

真正阻止读取，需要：

```text
Sandbox
Permission
Container isolation
Secret management
```

这就是：

```text
.gitignore ≠ security
```

非常常见的面试/实际工程问题。

---

# Agent 安全修改代码的最小工作流

今天开始建议形成这个习惯：

```text
① git status
      ↓
② 确认当前branch
      ↓
③ Agent开始修改
      ↓
④ git status
      ↓
⑤ git diff
      ↓
⑥ build/test
      ↓
⑦ 再看git diff
      ↓
⑧ review
      ↓
⑨ commit
```

并行任务则升级成：

```text
main
 │
 ├── worktree Agent-A
 │        ↓
 │       diff
 │        ↓
 │       test
 │
 └── worktree Agent-B
          ↓
         diff
          ↓
         test
```

这就是后面企业 Agent 平台的最初雏形。

## 今天的故障排查案例

如果 Agent 明明运行在：

```text
D:\sample-app-feature
```

UI 却显示：

```text
branch = main
```

不要马上重新启动模型。

依次检查：

```powershell
Get-Location
git rev-parse --show-toplevel
git branch --show-current
git rev-parse --git-dir
git rev-parse --git-common-dir
git worktree list
```

2026 年 OpenCode 就有公开 issue：打开 worktree 后 UI 显示主仓库 branch，而实际在 worktree 执行 `git rev-parse --abbrev-ref HEAD` 得到的 branch 是正确的。citeturn0search8

这就是标准的：

```text
UI state
≠
Git真实状态
```

排错时应优先相信可验证的底层事实。

---

## 企业岗位能力映射

今天实际上已经开始训练 **Git、workspace isolation、CWD、branch、diff、worktree、Agent 并发和故障定位**。

企业 Agent 平台的很多高级能力，本质都是今天这个概念继续扩展：

```text
Git worktree
     ↓
独立workspace
     ↓
Docker container
     ↓
独立filesystem/process/network
     ↓
Kubernetes Pod
     ↓
企业级Agent workload isolation
```

所以后面讲 Docker 时，你不会突然面对一个毫无关系的新技术，而会发现：

> **Git worktree 主要解决代码状态隔离；Container 进一步解决运行环境隔离。**

### 今日检查题

1. 为什么两个并行 Coding Agent 只创建不同 branch，还不一定足够？
2. `git status` 和 `git diff` 分别解决什么问题？
3. Agent 位于 worktree，但 Runtime 的 CWD 错误指向主 repository，会发生什么？
4. `.gitignore` 里写了 `.env`，为什么仍不能认为 API Key 安全？
5. Worktree 与 Docker 最大的区别是什么？

参考答案核心是：**worktree 给每个 Agent 独立 working directory；status 看状态、diff 看具体修改；CWD 错会操作错误工作区；gitignore 不阻止 Agent 读取文件；worktree 隔离代码工作区，而容器进一步隔离进程、依赖、文件系统、网络和资源。**

下一课沿着今天的真实操作继续进入 **Git staging / commit / branch / merge / conflict + Agent 自动修改安全机制**，并开始解释为什么 Agent 特别容易制造 merge conflict，以及 Codex/OpenCode 如何围绕 Git 状态建立执行环境。