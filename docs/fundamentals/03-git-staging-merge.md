## 第 3 课：Git staging / commit / merge conflict——让 Agent 的代码修改“可审查、可回滚、可合并”

今天解决一个非常实际的问题：**Agent 已经会修改代码了，但怎样避免它“一把梭”把不该提交的文件、错误修改、甚至密钥一起提交？**

核心答案是：

**Working Tree → Staging Area → Commit → Merge/PR**

Git 官方把 `index` 也就是 staging area 定义为“准备下一次 commit 内容的地方”；普通 `git commit` 只会记录已经 staged 的变化。citeturn143430search4turn143430search8

先建立这个心智模型：

```text
HEAD
 │
 │  上一次提交
 ▼
Repository
 │
 │
 ├───────────────┐
 │               │
 ▼               ▼
Index         Working Tree
暂存区          当前文件
```

你修改代码后：

```text
Working Tree:
Program.cs 已修改

Index:
还没有修改

HEAD:
旧版本
```

执行：

```bash
git add Program.cs
```

变成：

```text
Working Tree:
Program.cs 新版本

Index:
Program.cs 新版本

HEAD:
旧版本
```

再：

```bash
git commit -m "fix timeout"
```

才真正形成新的版本。

---

### 为什么 Agent 特别需要 staging area

假设 Coding Agent 完成任务后，项目状态是：

```text
modified:
src/App.cs

modified:
src/HttpClient.cs

modified:
README.md

untracked:
.env

untracked:
debug.log
```

如果 Agent 直接：

```bash
git add .
```

可能把：

```text
.env
debug.log
```

也放入 staging area。

其中 `.env` 甚至可能包含：

```text
OPENAI_API_KEY=...
DATABASE_PASSWORD=...
```

因此企业 Agent 不应该把：

```text
git add .
```

当成默认提交策略。

更安全的是：

```text
Agent完成代码修改
       ↓
git status
       ↓
检查 changed files
       ↓
git diff
       ↓
测试
       ↓
只 stage 允许的文件
       ↓
git diff --cached
       ↓
再次审查
       ↓
commit
```

这里第一次遇到：

```bash
git diff
```

和：

```bash
git diff --cached
```

Git 官方文档说明，普通 `git diff` 主要看 **working tree 相对 staging area 的变化**；`git diff --cached` 则可用来检查已经进入 index、准备提交的内容。citeturn143430search12

所以以后你可以把：

```text
git diff
```

理解成：

> “还有哪些修改没 stage？”

而：

```text
git diff --cached
```

理解成：

> “如果现在 commit，到底要提交什么？”

后者对 Coding Agent 特别重要。

---

## 一个企业 Agent 的正确提交闭环

以后我们的 Mini Agent Platform 会近似这样设计：

```python
changes = git_status()

safe_files = policy_filter(changes)

for file in safe_files:
    git_add(file)

staged_diff = git_diff_cached()

decision = reviewer.review(staged_diff)

if decision.safe:
    run_tests()

if tests_pass:
    git_commit()
```

注意这里有 **两个 Agent 角色**：

```text
Coder Agent
    ↓
修改代码

Reviewer Agent
    ↓
审查 staged diff
```

以后进入 multi-agent 时，这就是最简单的一种：

```text
Writer
+
Reviewer
```

结构。

---

# Claude Code 为什么也强调 Git diff

Claude Code 官方目前明确支持直接使用 Git，可以 staging、生成 commit message、创建 branch 和 PR。citeturn372191search19

更有意思的是，Anthropic 官方的 Skills 示例专门展示了：

> 获取 repository 当前真实 diff，然后让 Claude 总结未提交变化并标记风险。

这样做的意义是让 Agent 依据 **实际 working tree**，而不是凭上下文猜测“自己刚才改过什么”。citeturn143430search27

也就是说：

```text
LLM memory
     ↓
可能不准确

git diff
     ↓
实际状态
```

这又出现一个以后非常重要的 Agent 工程原则：

> **能通过工具获得事实，就不要让模型依赖记忆猜事实。**

---

# Codex 中 Git 为什么会和 Sandbox 冲突

现在进入一个非常典型的部署工程问题。

执行：

```bash
git add src/App.cs
```

看起来只是“暂存文件”。

实际上 Git 还需要修改 repository metadata，例如：

```text
.git/index
```

或者 linked worktree 中的：

```text
main-repo/.git/worktrees/<worktree>/index
```

并创建类似：

```text
index.lock
```

这样的锁文件。

2026 年 Codex 的公开 issue 中就出现了 linked worktree 下 `git add` 失败：sandbox 允许写 worktree 文件，却没有允许写主 repository 下对应的 `.git/worktrees/.../index.lock`。citeturn143430search22

于是：

```text
Agent：
git add app.py

↓
文件本身可写

↓
Git需要写：
.git/worktrees/agent/index.lock

↓
Sandbox禁止

↓
git add失败
```

这非常关键。

看到：

```text
Unable to create index.lock
Permission denied
```

不要马上判断：

> Git 坏了。

也不要判断：

> Agent 不会 Git。

应该检查：

```text
Git metadata path
sandbox writable roots
worktree类型
文件权限
```

Codex 在 Windows 上也出现过类似问题：workspace 本身可写，但 `.git` 写入被 sandbox 拦截，导致 worktree、commit、merge 等操作反复需要人工批准。citeturn143430search6

这就是企业岗位里的：

**应用故障 ≠ 模型故障。**

---

# Approval 与 Git side effect

现在比较：

```bash
git diff
```

和：

```bash
git push
```

它们风险完全不同。

`git diff`：

```text
只读
本地
几乎无副作用
```

`git push`：

```text
写远程repository
影响别人
可能触发CI/CD
甚至触发部署
```

因此企业 Agent 权限策略一般应该不同。

Claude Code 官方权限系统支持细粒度规则；官方示例里就能分别控制类似：

```text
Bash(git diff *)
Bash(git push *)
```

这样的命令。citeturn143430search3turn143430search23

你可以把策略想成：

```text
git status    → Allow
git diff      → Allow
git log       → Allow

git add       → Ask / Policy controlled
git commit    → Ask
git push      → Ask
git reset --hard → Deny / Strong Ask
```

这比单纯：

```text
Allow Shell
```

安全得多。

因为 Shell 并不是一个风险等级。

Shell 里面可能是：

```bash
pwd
```

也可能是：

```bash
rm -rf /
```

还可能是：

```bash
git push --force
```

---

# Kimi Code 的同类问题

Kimi CLI 生态中也已经出现针对 granular auto-approval 的需求，例如：

```text
git * → allow
rm *  → ask
```

这说明实际 Coding Agent 使用中，“Shell 是否允许”已经不够，而需要进入 **命令级策略**。citeturn372191search3

这里你应该开始建立一个企业安全模型：

```text
Tool
 ↓
Command
 ↓
Arguments
 ↓
Target resource
 ↓
Side effect
 ↓
Approval policy
```

例如：

```text
Shell
 ↓
git
 ↓
push
 ↓
origin/main
 ↓
remote write
 ↓
human approval
```

这就是以后 RBAC、Policy Engine、OPA 等概念的前身。

---

# Commit 到底是什么

很多初学者把：

```bash
git commit
```

理解成：

> “保存一下文件。”

不准确。

更准确是：

> 为当前 staging area 创建一个新的 Git 历史节点。

例如：

```text
A -- B -- C
```

你提交：

```bash
git commit -m "fix timeout"
```

变成：

```text
A -- B -- C -- D
```

其中 D 包含：

```text
父commit
tree快照
author
时间
commit message
```

所以 Coding Agent 做 commit 的价值并不是“让修改生效”。

真正价值是：

```text
形成可追踪版本
便于review
便于rollback
便于merge
便于PR
```

---

# 为什么 Agent 自动 commit 要谨慎

假设 Agent 任务：

> 修复登录 timeout。

但它修改：

```text
auth.cs
http.cs
database.cs
README.md
Dockerfile
```

如果 Agent直接：

```bash
git commit -am "fix timeout"
```

你会得到一个语义混乱的 commit。

好的企业代码习惯要求：

```text
一个commit
≈
一个逻辑修改
```

所以 Agent 以后应该能够判断：

```text
这些修改是不是同一个任务？
```

必要时拆分：

```text
commit 1:
fix HTTP timeout

commit 2:
add regression test
```

这叫：

**atomic commits**

后面 CI/CD 和代码审查会非常依赖这种习惯。

---

# Merge 是什么

现在有：

```text
main:
A -- B -- C

agent:
       └── D -- E
```

Agent 完成任务以后，可以把 agent branch 合回 main。

Git 官方定义 `git merge` 是把已经发生分叉的提交历史合并到当前 branch。citeturn143430search0

理想情况：

```text
A -- B -- C ------- M
       \           /
        D -- E ----
```

M 是 merge commit。

---

# 为什么会产生 Merge Conflict

假设 main 修改：

```csharp
int timeout = 20;
```

Agent branch 修改同一行：

```csharp
int timeout = 30;
```

Git 无法自动知道：

> 到底应该 20 还是 30？

于是：

```text
CONFLICT
```

文件中可能出现：

```text
<<<<<<< HEAD
int timeout = 20;
=======
int timeout = 30;
>>>>>>> agent/timeout
```

这三个 marker 必须认识：

```text
<<<<<<<
=======
>>>>>>>
```

Git status 也会把这种路径显示为 unmerged 状态。citeturn143430search16

---

# Agent 为什么特别容易制造 Conflict

人的开发过程通常是：

```text
修改
commit
pull/rebase
继续开发
```

Agent 有时会：

```text
长时间执行
↓
修改几十个文件
↓
期间main已经发生变化
↓
最后一次性merge
```

冲突概率明显增加。

更糟糕的是，多 Agent：

```text
Agent A → 修改 API
Agent B → 修改 API
Agent C → 修改 API
```

都基于旧 commit。

于是出现：

```text
stale branch
```

也就是“分支基线已经过时”。

以后企业 Agent 调度器需要做：

```text
任务启动前：
记录 base SHA

任务结束前：
检查 main HEAD

如果main已变化：
rebase / merge / restart / human review
```

这已经是 Agent orchestration 的一部分。

---

# 一个简单的 Agent merge 策略

可以设计：

```text
Agent完成
   ↓
git status必须clean except task changes
   ↓
run tests
   ↓
commit
   ↓
fetch latest main
   ↓
尝试merge/rebase
   ↓
没有冲突
   ├─ test
   └─ PR
   ↓
有冲突
   ↓
进入Conflict Resolver
   ↓
再次测试
   ↓
人工review
```

注意：

**能自动解决 conflict ≠ 应该永远自动解决。**

如果冲突涉及：

```text
数据库migration
安全权限
支付逻辑
生产配置
deployment
```

最好升级人工审批。

---

# Patch 与直接改文件

Git 官方还有：

```bash
git apply
```

它可以应用 patch，但不会自动创建 commit。citeturn143430search20

Coding Agent 经常更偏向：

```text
生成patch
↓
apply
↓
验证diff
```

而不是：

```text
直接把整个文件重写一遍
```

原因是 patch 更容易：

```text
审查
追踪
限制修改范围
检测context mismatch
```

以后我们会真正自己实现：

```text
read_file
↓
generate patch
↓
apply patch
↓
verify diff
```

这是 Coding Agent 文件编辑器的核心之一。

---

# Hooks：从“希望 Agent 记得”变成“系统强制执行”

假设团队规定：

> 每次 Agent 修改 Python 文件后必须执行 formatter。

差的做法：

```text
Prompt:
“记得运行 black。”
```

因为模型可能忘。

更可靠的是：

```text
Hook:
文件修改结束
↓
自动执行 formatter
```

Claude Code 官方 Hooks 就是用来在生命周期节点运行确定性 shell 命令，例如自动检查或执行项目规则；Anthropic 明确说明 hooks 是用户定义的 shell command。citeturn143430search11

这说明：

```text
LLM instruction
```

与：

```text
deterministic automation
```

是两种东西。

企业系统里必须知道什么时候用哪一种。

例如：

```text
判断代码设计是否合理
→ LLM

确保每次都运行 lint
→ hook / CI
```

但要特别注意：Claude Code 官方警告 command hook 使用当前系统用户权限执行，可以修改或删除该用户能访问的文件。citeturn143430search7

所以：

```text
Hook ≠ 天然安全
```

---

# 今天的 PowerShell 实验

继续使用你的测试 repository。

先创建文件：

```powershell
"version 1" | Out-File agent-test.txt
```

查看：

```powershell
git status
git diff
```

stage：

```powershell
git add agent-test.txt
```

再：

```powershell
git status
git diff
git diff --cached
```

重点观察：

**为什么普通 `git diff` 现在变化了，而 `git diff --cached` 出现内容。**

然后：

```powershell
git commit -m "add agent test file"
```

再：

```powershell
git status
git log --oneline -3
```

接下来制造一个非常小的 conflict。

```powershell
git switch -c agent/conflict
"agent version" | Out-File agent-test.txt
git add agent-test.txt
git commit -m "agent change"
```

切回 main：

```powershell
git switch main
"main version" | Out-File agent-test.txt
git add agent-test.txt
git commit -m "main change"
```

然后：

```powershell
git merge agent/conflict
```

这次很可能得到：

```text
CONFLICT
```

执行：

```powershell
git status
Get-Content agent-test.txt
```

找：

```text
<<<<<<<
=======
>>>>>>>
```

**今天不需要追求熟练解决 conflict。**

只要真正看到一次即可。

---

# 一个企业级故障排查案例

假设日志：

```text
Agent:
git add src/app.ts

Error:
fatal: Unable to create '.git/index.lock':
Permission denied
```

排查不要直接删除 `.git/index.lock`。

先问：

```text
① 当前cwd是什么？
② 是普通repo还是linked worktree？
③ git rev-parse --git-dir 输出什么？
④ git rev-parse --git-common-dir 输出什么？
⑤ sandbox是否允许写那里？
⑥ 是否真的存在另一个Git进程持有锁？
```

这里尤其重要：

> **看到 lock 文件错误，不等于 lock 文件一定是残留垃圾。**

可能实际上是：

```text
Sandbox denied write
```

2026 年 Codex 的 linked-worktree 问题就是这样的典型案例。citeturn143430search22

这就是工程师的排错方式：

```text
现象
↓
验证假设
↓
定位layer
```

而不是：

```text
看到报错
↓
Google一个删除命令
↓
直接执行
```

---

## 今天需要形成的 Agent 安全模型

把前 3 课连起来，现在一个成熟的 Coding Agent 最低应该有：

```text
User Task
   ↓
LLM
   ↓
Tool Call
   ↓
Permission Policy
   ↓
Git Worktree
   ↓
File Edit / Patch
   ↓
git status
   ↓
git diff
   ↓
Selective staging
   ↓
git diff --cached
   ↓
Test
   ↓
Review
   ↓
Commit
   ↓
Merge / PR
```

再往后才是：

```text
Docker
CI/CD
Kubernetes
Deployment
```

也就是说：

**Git 是 Agent 从“会改代码”进入“工程化改代码”的第一道基础设施。**

---

## 今天对应的企业岗位能力

你今天已经接触到：

**Git index/staging、atomic commit、merge conflict、patch、approval、hooks、sandbox 与 Git metadata。**

这些内容以后对应的真实工作包括：

```text
Coding Agent平台开发
DevOps
CI/CD
代码审查自动化
Agent安全治理
多Agent并发workspace
故障排查
```

### 今日检查题

1. 为什么 `git add .` 对自动化 Agent 风险比较高？
2. `git diff` 与 `git diff --cached` 最大区别是什么？
3. 为什么 Codex 在 worktree 中可能“文件能写，但 git add 失败”？
4. Merge conflict 本质上为什么产生？
5. 为什么 lint 更适合 Hook/CI，而“判断架构设计是否合理”更适合 LLM？

参考答案核心：

```text
1. 容易stage不属于任务甚至敏感文件。
2. 一个主要看未stage修改，一个检查准备commit的staged修改。
3. Git还需要写.git/worktrees等metadata，而sandbox可能没授权。
4. 不同branch修改了Git无法自动决定如何合并的内容。
5. lint是确定性规则；设计判断需要语义推理。
```

**下一课进入 Bash / PowerShell 的真正核心：进程、管道、重定向、环境变量、PATH、cwd，以及为什么同一条 Agent 命令在 Windows、macOS、Linux、Docker 中会产生完全不同的结果。**