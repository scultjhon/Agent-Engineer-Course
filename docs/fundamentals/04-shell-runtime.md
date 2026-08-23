# 第 4 课：Bash / PowerShell 与 Agent Runtime——为什么同一条命令在不同环境里表现完全不同

今天开始补真正支撑 Agent 的操作系统基础。目标不是学一堆 Shell 语法，而是看懂 **Agent 到底怎样启动进程、传环境变量、接收 stdout/stderr、处理超时和当前目录**。

这部分非常重要，因为大量“Agent 失败”其实不是模型问题，而是 **进程和执行环境问题**。

先记住 Agent 调用 Shell 的真实结构：

```text
LLM
 ↓
tool_call
 ↓
Agent Runtime
 ↓
创建子进程
 ↓
shell / executable
 ↓
stdin
 ↓
process
 ↓
stdout / stderr / exit code
 ↓
Runtime
 ↓
重新喂给模型
```

Codex 的公开 issue 就有一个很典型的问题：命令返回非 0 时，如果 stdout/stderr 没有正确回传，模型就看不到 lint、编译或测试的具体错误，自然无法继续修。citeturn992668search0

所以企业 Agent 工程里，**“命令失败”还不够，必须把失败信息完整带回来。**

---

## 1. `cwd`：Agent 到底在哪个目录执行

`cwd` 是：

```text
Current Working Directory
```

也就是当前工作目录。

比如：

```powershell
Get-Location
```

或者 Bash：

```bash
pwd
```

如果 Agent 想执行：

```bash
dotnet test
```

但 cwd 错了：

```text
正确：
C:\Projects\sample-app

实际：
C:\
```

结果可能就是：

```text
Could not find a project
```

这并不是：

```text
dotnet坏了
模型不会C#
```

而只是：

```text
cwd错误
```

OpenCode 2026 年就有公开 issue：Desktop 版本工具被错误地从 `/` 根目录执行，而不是项目目录，导致相对路径工具全部失效。citeturn992668search17

所以以后排查 Agent 命令问题，第一个问题之一就是：

```text
当前 cwd 是什么？
```

---

## 2. 为什么 `cd` 经常让 Agent 初学者困惑

假设 Agent 第一次调用：

```bash
cd src
```

然后下一次调用：

```bash
pwd
```

你可能以为应该还是：

```text
/project/src
```

但不一定。

因为很多 Agent 的 Shell Tool 每次都会：

```text
创建新的 subprocess
```

因此：

```text
Tool Call #1
shell process A
cd src
进程结束

Tool Call #2
shell process B
pwd
```

进程 B 根本不知道 A 执行过 `cd`。

Kimi CLI 2026 年的一个公开 issue 就明确涉及这个问题：Shell tool 每次启动新的 subprocess，因此单独执行 `cd` 只影响那个子进程。citeturn992668search22

所以：

```bash
cd src
```

再单独运行：

```bash
dotnet test
```

可能失败。

更可靠的是：

```bash
cd src && dotnet test
```

或者 Runtime 本身设置：

```text
cwd=/project/src
```

这就是为什么 Agent Runtime 必须有：

```text
working_directory
```

概念。

---

## 3. stdin / stdout / stderr

一个进程有三个最重要的标准流：

```text
stdin  → 输入
stdout → 正常输出
stderr → 错误/诊断输出
```

比如：

```bash
git status
```

输出一般进入：

```text
stdout
```

而错误：

```bash
git abc
```

可能进入：

```text
stderr
```

Agent Runtime 通常做类似事情：

```python
result = subprocess.run(
    command,
    capture_output=True,
    text=True
)
```

然后得到：

```python
result.stdout
result.stderr
result.returncode
```

C# 类比就是：

```csharp
Process process = new Process();

process.StartInfo.RedirectStandardOutput = true;
process.StartInfo.RedirectStandardError = true;
```

所以如果你以前做过 `.NET Process`，其实已经接触过 Agent Shell Tool 的底层基础。

---

## 4. stdout/stderr 为什么会造成 Agent 卡死

这个问题非常工程化。

OpenCode 2026 年出现过 Bash Tool 卡死问题：

```bash
bash -c "sleep 10 &"
```

看上去：

```text
父shell已经退出
```

但后台子进程还继承了：

```text
stdout
stderr
```

对应的 pipe 文件描述符。

Node.js 运行时可能继续等待这些流真正关闭，于是：

```text
Agent一直认为命令没结束
```

哪怕 shell 本身早就退出了。citeturn992668search13turn992668search1

这是很典型的：

```text
process lifetime
≠
stdio lifetime
```

问题。

企业 Agent 部署工程师必须能看懂这种现象。

---

# 5. 什么是 pipe

Shell 中：

```bash
commandA | commandB
```

意思是：

```text
commandA stdout
       ↓
      pipe
       ↓
commandB stdin
```

例如：

```bash
git status --short | grep ".cs"
```

可以理解成：

```text
git status --short
        ↓ stdout
       pipe
        ↓ stdin
grep ".cs"
```

Windows PowerShell 也有：

```powershell
Get-ChildItem | Where-Object Name -Like "*.cs"
```

但注意：

**Bash 的 pipe 主要传文本流；PowerShell 的 pipeline 经常传对象。**

这是二者非常大的思维区别。

例如 PowerShell：

```powershell
Get-Process | Where-Object CPU -gt 100
```

中间传的不只是字符串，而可能是：

```text
Process Object
```

所以 PowerShell 不应该简单理解成“Windows 版 Bash”。

---

# 6. 重定向

Bash：

```bash
echo hello > out.txt
```

意思是：

```text
stdout
 ↓
out.txt
```

追加：

```bash
echo hello >> out.txt
```

stderr：

```bash
command 2> error.log
```

合并 stdout + stderr：

```bash
command > output.log 2>&1
```

这在 Agent 场景特别常见。

例如：

```bash
dotnet test > test.log 2>&1
```

然后 Agent 可以：

```bash
cat test.log
```

分析。

不过现代 Agent Runtime 更推荐直接捕获进程输出，不一定必须经由文件。

---

# 7. PowerShell 对应写法

PowerShell 也支持：

```powershell
dotnet test > test.log
```

以及：

```powershell
dotnet test 2> error.log
```

查看：

```powershell
Get-Content test.log
```

但 Agent 经常直接让 Runtime 捕获：

```text
stdout
stderr
```

而不落磁盘。

这里要培养一个习惯：

> 当你看到 Agent 卡在“Executing command...”时，要考虑它究竟是在等进程退出，还是在等 stdout/stderr pipe 关闭。

---

# 8. stdin 为什么也会把 Agent 卡住

假设 Agent 执行：

```bash
git push
```

Git 发现需要用户名密码，于是：

```text
Username:
```

它等待：

```text
stdin
```

但是 Agent Runtime 并没有人工终端。

结果就是：

```text
一直卡着
```

Kimi CLI 最近的 changelog 就专门修过这一类问题：

- Shell tool 会关闭 stdin；
- 设置 `GIT_TERMINAL_PROMPT=0`；
- 需要凭据的 Git 命令不再无限等待，而是尽快失败并把错误返回。citeturn992668search2

这就是很优秀的 Agent Runtime 设计。

从：

```text
等待人工输入
```

变成：

```text
fail fast
↓
stderr
↓
LLM判断
```

---

# 9. 环境变量

你会经常看到：

```text
OPENAI_API_KEY
ANTHROPIC_API_KEY
PATH
HTTP_PROXY
HTTPS_PROXY
CUDA_VISIBLE_DEVICES
```

这些都属于：

```text
environment variables
```

Bash：

```bash
export MY_VAR=hello
echo $MY_VAR
```

PowerShell：

```powershell
$env:MY_VAR="hello"
$env:MY_VAR
```

Agent Runtime 启动子进程时，会把环境变量传进去。

因此：

```text
终端里命令能运行
Agent里却不行
```

一个很重要的原因就是：

```text
environment不同
```

例如你自己的 PowerShell：

```text
PATH里有：
C:\Program Files\dotnet
```

但 Docker 或 Agent sandbox 里：

```text
PATH里没有
```

于是：

```text
dotnet: command not found
```

---

# 10. PATH 到底是什么

假设 Agent 执行：

```bash
git
```

系统并不会凭空知道 `git.exe` 在哪里。

它会按照 PATH 逐个寻找。

Windows 可能类似：

```text
C:\Windows\System32
C:\Program Files\Git\cmd
C:\Program Files\dotnet
...
```

Linux/macOS：

```text
/usr/local/bin
/usr/bin
/bin
...
```

查看：

Bash：

```bash
echo $PATH
```

PowerShell：

```powershell
$env:PATH
```

寻找命令：

Bash：

```bash
which git
```

PowerShell：

```powershell
Get-Command git
```

如果 Agent 报：

```text
git not found
```

工程排查顺序应该是：

```text
git是否安装
↓
PATH是否包含
↓
Agent env与用户shell是否一致
↓
sandbox/container是否有git
```

---

# 11. Codex 的一个非常真实 Windows Bug

2026 年 Codex 在 Windows 上有一个很有代表性的 issue：

内部为了终止进程树，会调用：

```text
taskkill /T /F /PID ...
```

但 `taskkill` 的 stdout 被意外泄漏到 Codex 自己的 stdout。

更麻烦的是，不同语言 Windows 输出可能不同。

例如中文系统输出中文。

下游工具本来期待：

```text
纯 JSONL
```

结果中间突然混入操作系统文本，于是 parser 直接崩。citeturn992668search12

这里暴露了一个重要企业问题：

```text
stdout 是接口
```

如果某个程序承诺：

```text
stdout = JSONL协议
```

那么调试文字就应该放：

```text
stderr
```

否则会污染协议。

这是 CLI 工程和 Agent 平台非常重要的规范。

---

# 12. 为什么 `codex exec` 在 CI 环境可能表现不同

另一个 2026 年 Codex issue 显示：

```text
有TTY
→ 正常

没有controlling TTY
→ 可能异常退出
```

尤其是在：

```text
CI
background process
daemon
subprocess
```

环境中更加明显。citeturn992668search8

这涉及：

```text
TTY
PTY
non-interactive mode
```

今天只需要先记：

```text
有终端
≠
无终端
```

很多 CLI 工具会检测：

```text
isatty()
```

然后改变行为。

所以：

```text
我在终端运行正常
GitHub Actions里面失败
```

非常常见。

后面 CI/CD 课程会专门讲。

---

# 13. timeout 有两层

假设：

```bash
timeout 10 myprogram
```

外面 Agent Runtime 自己又设置：

```text
tool timeout = 5秒
```

结果：

```text
外层5秒先杀掉
```

模型看到：

```text
Tool timed out
```

而不是程序原本想返回的：

```text
exit code 124
```

Codex 之前就有公开 issue 讨论这种“tool timeout 抢先终止 inner timeout”的情况。citeturn992668search4

所以以后看到 timeout：

不要只问：

```text
超时了吗？
```

而要问：

```text
是哪一层 timeout？
```

可能是：

```text
LLM API timeout
Agent tool timeout
shell timeout
application timeout
HTTP timeout
database timeout
Kubernetes deadline
```

企业排错经常就是不断确定：

```text
是哪一层
```

---

# 14. Kimi / OpenCode / Codex 对比

今天可以先做这样一个工程化比较：

|问题|Codex|OpenCode|Kimi Code|
|---|---|---|---|
|Shell执行|Rust runtime/tool|Bash tool|Shell tool|
|cwd问题|有独立runtime/workspace概念|Desktop曾有cwd bug|Shell调用同样依赖工作目录|
|stdout/stderr|必须正确回传模型|后台进程继承pipe可能卡住|近期修复stdin交互阻塞|
|timeout|tool层超时曾出现语义问题|background process可能影响结束判断|支持避免交互命令无限等待|
|Windows细节|taskkill/stdout等实际bug很多|Desktop与CLI环境差异|持续修Windows subprocess问题|

这些不是“哪个工具差”。

恰恰说明：

> Coding Agent 本质上是一个复杂的本地执行系统。

---

# 15. 企业 Agent 的 Shell Executor 应该长什么样

以后自己写 Mini Agent 时，我们不会简单：

```python
os.system(command)
```

而应该逐渐变成：

```python
class ShellResult:
    stdout: str
    stderr: str
    exit_code: int
    timed_out: bool
    duration_ms: int

async def execute(
    command,
    cwd,
    env,
    timeout
) -> ShellResult:
    ...
```

甚至还要增加：

```text
max_output_bytes
allowed_commands
sandbox
network_policy
stdin_policy
kill_process_tree
```

最终：

```text
LLM
 ↓
Shell Tool
 ↓
Policy
 ↓
Sandbox
 ↓
Process Manager
 ↓
stdout/stderr
 ↓
Trace
```

这就是企业 Agent Runtime。

---

# 今日 10 分钟实验

Windows PowerShell：

```powershell
Get-Location
Get-Command git
$env:PATH
```

然后：

```powershell
git --version
$LASTEXITCODE
```

再试一个错误命令：

```powershell
git abcxyz
$LASTEXITCODE
```

观察：

```text
stdout
stderr
exit code
```

再：

```powershell
Set-Location ..
Get-Location
```

注意 cwd 变化。

然后测试子进程工作目录：

```powershell
powershell -Command "Set-Location ..; Get-Location"
Get-Location
```

你会发现：

```text
子PowerShell改变目录
```

不会永久改变：

```text
父PowerShell目录
```

这就是 Agent 每次启动新 subprocess 时 `cd` 失效问题的最小实验。

macOS/Linux：

```bash
pwd
which git
echo $PATH
```

然后：

```bash
bash -c 'cd ..; pwd'
pwd
```

同样观察：

```text
child process cwd
≠
parent process cwd
```

---

# 今日企业故障案例

现象：

```text
本地：
npm test
成功

Agent：
npm test
command not found
```

不要重新 Prompt。

正确故障树：

```text
Agent执行失败
 │
 ├─ cwd正确？
 │
 ├─ npm安装了吗？
 │
 ├─ PATH是否相同？
 │
 ├─ Agent使用哪个shell？
 │
 ├─ sandbox内能看到node吗？
 │
 ├─ Docker镜像里有没有npm？
 │
 └─ CI环境是否不同？
```

如果最终发现：

```text
宿主机Node安装在：
/opt/homebrew/bin/node

Agent container：
PATH没有/opt/homebrew/bin
```

那么这就是：

```text
runtime environment问题
```

而不是：

```text
LLM问题
```

---

## 岗位能力映射

今天开始进入真正的 **Agent Runtime / Deployment Engineer** 技能：

`process、cwd、env、PATH、stdin/stdout/stderr、pipe、redirect、timeout、TTY、process tree`。

后面 Docker、Kubernetes、CI/CD 其实都会不断重复今天这些概念。例如 Docker 的：

```text
WORKDIR
ENV
ENTRYPOINT
STDOUT日志
signal
PID 1
```

本质都和今天紧密相关。

### 今日检查题

1. 为什么 Agent 单独执行一次 `cd src`，下一次 Tool Call 不一定还在 `src`？
2. 为什么后台进程即使父 Shell 已经退出，Agent Tool 仍可能“卡住”？
3. `PATH` 对 Agent 有什么作用？
4. 为什么企业 CLI 应该把协议数据放 stdout、诊断信息放 stderr？
5. “命令超时”为什么不能直接说明应用本身超时？

答案核心：**Shell Tool 很可能每次是新子进程；后台进程可能继续持有 stdout/stderr pipe；PATH 决定可执行程序搜索路径；stdout 可能是机器接口，stderr 应承载诊断；超时可能发生在 LLM、Agent Tool、Shell 或应用的不同层。**

下一课进入 **Linux 进程、PID、信号、前后台任务和进程树**，直接解释 Agent 为什么需要 kill process tree，以及 Docker/Kubernetes 为什么特别关心 PID 1。