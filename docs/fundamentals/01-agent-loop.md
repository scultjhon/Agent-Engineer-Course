## 第 1 课：Coding Agent 到底是怎么“动手干活”的

今天先建立最重要的工程直觉：**Codex、Kimi Code、OpenCode、Claude Code 不是“模型自己在操作电脑”，而是模型负责决策，Agent Runtime 负责真正执行命令、读写文件、返回结果。**

你后面能不能真正调试 Agent，关键就在于能否看清这条链路。

```text
用户需求
  ↓
Agent Runtime
  ↓
构造上下文 / system prompt / tools
  ↓
LLM
  ↓
返回普通文本 或 Tool Call
  ↓
Runtime 判断权限
  ↓
调用 Shell / Read / Write / Patch / MCP
  ↓
操作系统真正执行
  ↓
stdout / stderr / exit code
  ↓
结果重新送给 LLM
  ↓
LLM 决定下一步
  ↓
测试 / 修改 / 再测试
  ↓
最终回复
```

这就是以后反复出现的 **Agent Loop**。

### 1. 先纠正一个很重要的认识

例如 Claude 官方的 Bash Tool 文档明确说明：**Claude 本身并不运行命令**。模型返回一个 `tool_use`，其中带着类似 `ls *.py` 的命令；真正启动 Bash、维护进程、设置超时、做安全检查的是你的 Agent 应用。命令执行后的 stdout/stderr 再通过 `tool_result` 返回给 Claude。citeturn160416view3

所以以后遇到：

```text
Claude 执行命令失败
```

不要马上得出：

```text
Claude 模型能力不行
```

真正的故障点可能有：

```text
LLM生成命令错误
Shell不存在
PATH不对
当前目录不对
权限不足
Sandbox拦截
依赖没安装
进程超时
stderr被截断
Runtime错误
Tool结果没正确回传模型
```

这就是 **Agent 工程师与普通 Agent 用户的区别**。

---

## 2. 用真实源码看 Kimi Code 的结构

Kimi CLI 的项目文档已经把核心架构暴露得非常清楚。

核心 Agent Loop 在：

```text
src/kimi_cli/soul/kimisoul.py
```

负责：

```text
用户输入
→ Context
→ LLM
→ Tool Call
→ KimiToolset执行
→ Context更新
→ 再调用LLM
```

工具调度在：

```text
src/kimi_cli/soul/toolset.py
```

Shell、文件、Web、Agent 等真正的工具在：

```text
src/kimi_cli/tools/
```

而且现在还包括：

```text
agent
shell
file
web
todo
background
think
plan
```

Subagent 又有：

```text
src/kimi_cli/soul/agent.py
```

以及持久化、context、compaction、approval 等模块。Kimi 自己的 `AGENTS.md` 就给出了这张源码地图。citeturn155859search6

可以先把它理解为 C#：

```csharp
class KimiSoul
{
    Context context;
    Toolset tools;
    LlmClient llm;

    async Task Run(string userInput)
    {
        context.Add(userInput);

        while (true)
        {
            var response = await llm.Call(context, tools);

            if (response.ToolCall != null)
            {
                var result = await tools.Execute(response.ToolCall);
                context.Add(result);
                continue;
            }

            Console.WriteLine(response.Text);
            break;
        }
    }
}
```

真实实现当然复杂得多，但**骨架基本就是这个意思**。

这段伪代码以后一定要记住。

---

# 3. 第一个 Shell 基础：什么是进程

假如 Agent 生成：

```bash
git status
```

Agent Runtime 实际做的事情类似：

```text
Agent进程
   │
   ├─ 创建子进程
   │
   └─ git status
         │
         ├─ stdout
         ├─ stderr
         └─ exit code
```

这里第一次出现三个必须掌握的词：

```text
stdout
stderr
exit code
```

例如：

```bash
git status
```

执行成功：

```text
stdout:
On branch main
nothing to commit

stderr:
空

exit code:
0
```

执行不存在的命令：

```bash
abcxyz
```

可能得到：

```text
stderr:
command not found

exit code:
127
```

于是 Agent Runtime 可以告诉模型：

```json
{
  "command": "abcxyz",
  "exit_code": 127,
  "stderr": "command not found"
}
```

模型看到失败后：

```text
“命令不存在，我换一种方法。”
```

然后继续 Agent Loop。

---

# 4. 为什么 exit code 特别重要

通常约定：

```text
0 = 成功

非 0 = 失败或特殊结果
```

例如：

```bash
dotnet test
```

测试通过：

```text
exit code = 0
```

测试失败：

```text
exit code != 0
```

所以一个 Coding Agent 可以简单地实现：

```python
result = run("dotnet test")

if result.returncode != 0:
    context.add(result.stderr)
    ask_model_to_fix()
```

这就产生了你经常看到的：

```text
Codex修改代码
↓
运行测试
↓
测试失败
↓
查看错误
↓
修改代码
↓
再次测试
```

不是模型具有某种神秘的“自愈能力”。

本质就是：

**反馈闭环。**

---

# 5. Codex 的真实例子

Codex 当前公开的 Rust 实现里，App Server 已经把一次 Agent 工作明确组织成：

```text
Thread
  ↓
Turn
  ↓
Item
```

例如：

```text
thread/start
turn/start
item/started
item/commandExecution/outputDelta
item/completed
turn/completed
```

其中甚至有一个公开示例：

```text
git status --short
```

执行过程中，命令输出以 `outputDelta` 流式返回。citeturn160416view2

所以 UI 中看到：

```text
Running git status...
```

然后文字一点点出现，背后不是“模型正在想”。

可能只是：

```text
子进程 stdout
      ↓
Runtime
      ↓
event
      ↓
TUI
```

这是 Agent 调试时非常重要的判断。

---

# 6. 为什么 Coding Agent 很依赖 Git

今天只掌握 Git 的第一个命令：

```bash
git status
```

以及：

```bash
git diff
```

Agent 修改代码以后最重要的问题不是：

> 文件有没有变化？

而是：

> **到底改了什么？**

于是典型流程是：

```text
git status
    ↓
知道哪些文件变化

git diff
    ↓
知道具体改了哪些行
```

例如：

```bash
git diff -- src/App.cs
```

Agent 可以看到：

```diff
- var timeout = 10;
+ var timeout = 30;
```

这就是以后要讲的：

```text
working tree
index
HEAD
commit
branch
worktree
patch
diff
```

为什么企业 Agent 大量使用 Git？

因为 Git 天然提供：

```text
变更检测
diff
版本历史
隔离
回滚
审查
协作
```

这恰好都是 Agent 自动修改代码最需要的能力。

---

# 7. OpenCode 又增加了一层：Agent 权限

OpenCode 当前公开实现自带：

```text
build agent
plan agent
```

`build` 可以真正开发；

`plan` 默认不能修改文件，而且运行 Bash 前需要请求权限。citeturn160416view1

于是出现一个非常重要的企业 Agent 概念：

```text
Agent能力
≠
Agent权限
```

模型可能完全知道：

```bash
rm -rf build/
```

应该怎么执行。

但 Runtime 可以决定：

```text
不允许。
```

或者：

```text
需要人工审批。
```

以后我们会逐渐实现：

```text
Allow
Ask
Deny
```

例如：

```text
git status        → Allow

dotnet test       → Allow

npm install       → Ask

git push          → Ask

rm -rf /          → Deny
```

---

# 8. Codex 为什么还需要 Sandbox

Codex 的公开实现甚至有专门的：

```text
codex-rs/linux-sandbox/
```

也就是说：

**“能不能调用 shell”与“shell 能访问什么”是两个问题。**

例如 Agent 能运行：

```bash
cat /etc/passwd
```

但 sandbox 可以阻止它读取。

Agent 能运行：

```bash
echo hello > /xxx/a.txt
```

但 sandbox 可以限制只能写：

```text
当前 workspace
```

Codex 的公开 App Server 文档中也有 `workspace-write` 等 sandbox 概念。citeturn160416view2turn155859search1

以后到了 Docker，你会发现：

```text
Sandbox
Container
Namespace
Filesystem isolation
Permission
```

其实全连接起来了。

---

# 9. 一个非常真实的 Agent 故障

比如未来你碰到：

```text
Codex:
Found no NVIDIA driver
```

普通用户可能认为：

> CUDA 配错了。

但 2026 年 Codex 的公开 issue 就出现过：

```text
宿主机：
torch.cuda.is_available()
→ True

Codex sandbox 内：
→ 找不到 NVIDIA
```

原因是 Linux sandbox 没把：

```text
/dev/nvidia*
```

映射进去。citeturn155859search17

这就是非常典型的 **Agent 部署工程师问题**。

模型没有错。

PyTorch 没错。

CUDA 甚至也可能没错。

真正问题在：

```text
Agent Sandbox
        ↓
Linux namespace / device mapping
        ↓
GPU不可见
```

以后学 Docker GPU、Kubernetes GPU 时还会再次遇到这个问题。

---

# 10. Bash 和 PowerShell 为什么都必须学

例如：

### Bash

```bash
export API_KEY=abc
echo $API_KEY
```

### PowerShell

```powershell
$env:API_KEY="abc"
$env:API_KEY
```

含义相同：

```text
设置环境变量
```

但语法完全不同。

又例如：

### Bash

```bash
ls | grep src
```

### PowerShell

```powershell
Get-ChildItem | Where-Object Name -Like "*src*"
```

因此模型如果把 Bash 命令错误地发给 PowerShell，就可能失败。

这不是模型一定“不聪明”，而是：

```text
Agent必须知道
OS + Shell + cwd + env
```

Kimi CLI 自己就曾涉及 Windows shell 选择问题；其代码路径曾明确区分 Windows PowerShell 等 shell 环境。citeturn155859search2

---

# 11. 5～10 分钟实验

你今天只做这几个命令即可。

在一个 Git 项目目录中运行：

```bash
git status
```

然后：

```bash
git diff
```

再执行一个必定成功的命令：

### PowerShell

```powershell
Write-Output "hello agent"
$LASTEXITCODE
```

再试：

```powershell
git --version
$LASTEXITCODE
```

如果使用 Bash：

```bash
echo "hello agent"
echo $?
```

以及：

```bash
git --version
echo $?
```

注意：

```text
PowerShell:
$LASTEXITCODE

Bash:
$?
```

都是查看上一个外部程序退出码的常见方式。

今天**不要背这些命令**。

你只需要形成：

```text
Agent
↓
启动程序
↓
程序输出 stdout/stderr
↓
返回 exit code
↓
Agent据此决定下一步
```

这个认知即可。

---

# 12. 今天第一次建立 Agent 故障树

以后看到：

```text
Agent执行失败
```

第一反应应该是：

```text
                Agent失败
                   │
      ┌────────────┼─────────────┐
      ↓            ↓             ↓
    模型层       Runtime层       OS层
      │            │             │
 prompt        tool schema      shell
 context       approval         PATH
 reasoning     sandbox          cwd
              timeout           permission
              parser            process
                                  │
                                  ↓
                              dependency
```

后面还会继续增加：

```text
Git
Docker
Network
Kubernetes
Model Server
Database
MCP
Secrets
GPU
```

最终你看到一个报错，应当能逐层定位，而不是一直换 Prompt。

---

## 与四种 Agent 的第一轮对比

| Agent | 今天先记住的特点 |
|---|---|
| Codex | Runtime、thread/turn/item、shell execution、sandbox 体系值得深入读源码 |
| Kimi Code | Python 源码结构非常适合作为 Agent Loop 教材，Soul/Context/Toolset/Approval/Subagent 划分清楚 |
| OpenCode | 很适合研究 build/plan agent、权限与多 Agent |
| Claude Code | 核心产品不按开源源码讲；用官方 Bash Tool、SDK、MCP、Skills 和实际行为理解机制 |

Kimi 的旧 `kimi-cli` 仓库目前也明确表示正在逐步迁移到新一代 **Kimi Code CLI**，所以后面的源码课会注意区分历史代码与当前实现，不拿旧结构冒充当前状态。citeturn160416view0

---

## 企业岗位对应能力

今天其实已经接触了 6 个企业工程能力：

**Agent Runtime、Tool Calling、Process、Shell、Git、Sandbox。**

这也是为什么这套课不会直接从“写 Prompt”开始。

对于企业 Agent 工程师，Prompt 只是其中一层。

真正的系统更接近：

```text
LLM
+
Agent Runtime
+
OS
+
Git
+
Shell
+
Security
+
Container
+
Network
+
Observability
```

### 今天的 3 道检查题

1. 模型返回 `git status`，到底是谁真正执行 `git.exe` / `git` 进程？
2. `dotnet test` 返回非 0 exit code，对 Coding Agent 有什么意义？
3. 一个 Agent 知道如何删除 `/etc`，是否等于它真的能够删除？

如果你能回答：

```text
1. Agent Runtime / Tool执行器
2. 表示命令/测试失败，Runtime可以把错误重新交给模型继续处理
3. 不等于，还受权限、approval、sandbox、OS权限等限制
```

那么今天这节已经达标。

**下一课进入 Git + Agent：为什么 Codex/Claude Code 修改代码前后如此依赖 `git status`、`git diff`、branch 和 worktree，并实际建立一个“Agent 安全修改代码”的 Git 工作流。**