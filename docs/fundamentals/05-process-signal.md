# 第 5 课：Linux 进程、PID、信号与进程树——为什么 Agent 会“命令结束了却还卡着”

今天解决一个非常典型的 Agent Runtime 问题：

> Agent 执行 `npm run dev`、`vite`、`uvicorn`、MCP Server 或后台脚本后，明明命令看起来已经结束，为什么 Codex / OpenCode / Kimi 还一直卡着？

这通常不是模型问题，而是 **process tree、pipe、signal、child process** 没处理好。

先建立今天的主线：

```text
LLM
 ↓
Shell Tool
 ↓
Agent Runtime
 ↓
启动 shell 进程
 ↓
shell 再启动 child
 ↓
child 还可能启动 grandchild
 ↓
形成 process tree
 ↓
stdout / stderr pipe
 ↓
超时或用户取消
 ↓
Runtime 必须正确终止整个进程树
```

OpenCode 在 2026 年就出现过非常典型的问题：shell 本身已经退出，但后台子进程继续持有 stdout/stderr 的 pipe，导致 Bash Tool 一直等待 EOF，从而“挂住”。citeturn360578search1turn360578search10 Kimi CLI 也有 detached child process 导致 Shell Tool 无法按预期返回的公开问题。citeturn360578search4

---

## 1. PID 是什么

Linux 每个运行中的进程都有一个：

```text
PID
Process ID
```

例如：

```bash
ps
```

可能看到：

```text
PID    COMMAND
1201   bash
1250   python
1308   node
```

可以把它理解成：

```text
操作系统给进程分配的编号
```

例如：

```text
PID 1201 → bash
PID 1250 → python
```

Agent Runtime 启动：

```bash
python app.py
```

操作系统可能创建：

```text
Agent Runtime
    │
    └── PID 1250 python app.py
```

Runtime 后面如果想取消任务，至少需要知道：

```text
1250
```

---

## 2. 但只记录 PID 还不够

比如 Agent 执行：

```bash
npm run dev
```

真正可能发生：

```text
Agent Runtime
    │
    └── npm
         │
         └── node
              │
              └── vite
```

形成：

```text
parent
 ↓
child
 ↓
grandchild
```

如果 Runtime 只杀掉：

```text
npm
```

那么：

```text
vite
```

可能仍然存活。

于是你会遇到：

```text
Agent说已经取消
↓
端口5173仍然占用
```

甚至下一次 Agent 再运行：

```bash
npm run dev
```

得到：

```text
Address already in use
```

因此一个成熟 Agent Runtime 需要处理：

```text
process tree
```

而不是只处理单 PID。

---

# 3. PPID

Linux 还有：

```text
PPID
Parent Process ID
```

例如：

```bash
ps -ef
```

可能看到：

```text
PID   PPID   CMD

1000     1   bash
1200  1000   npm run dev
1205  1200   node vite
```

于是可以恢复：

```text
bash
 └── npm
      └── node vite
```

这对排查 Agent 残留进程非常有用。

以后你看到：

```text
端口占用
后台任务未退出
Agent结束后还有python
MCP进程残留
```

就应该想到：

```text
PID
PPID
process tree
```

---

# 4. `ps`、`pgrep`、`pstree`

Linux/macOS 常见：

```bash
ps aux
```

或者：

```bash
ps -ef
```

找 Node：

```bash
ps -ef | grep node
```

更适合程序使用：

```bash
pgrep node
```

显示进程树：

```bash
pstree
```

可能：

```text
agent───bash───npm───node
```

这里再次用到上一课的 pipe：

```bash
ps -ef | grep node
```

就是：

```text
ps stdout
   ↓
pipe
   ↓
grep stdin
```

---

# 5. Windows 对应是什么

Windows PowerShell：

```powershell
Get-Process
```

找 Node：

```powershell
Get-Process node
```

结束：

```powershell
Stop-Process -Id 1234
```

Windows 还有：

```cmd
taskkill /PID 1234 /T /F
```

其中：

```text
/T
```

表示终止子进程树。

这正是 Coding Agent Runtime 经常需要的操作。

Codex 在 Windows 上的公开问题甚至暴露过内部 `taskkill /T /F /PID ...` 的输出意外混进 CLI stdout，污染机器可读协议的问题。换句话说，**进程树终止本身就是 Codex Runtime 的真实工程逻辑之一。** citeturn360578search22

---

# 6. Linux signal 到底是什么

结束进程不是简单：

```text
kill()
```

Linux 主要使用：

```text
signal
```

通知进程发生事件。

最需要掌握三个：

```text
SIGINT
SIGTERM
SIGKILL
```

先理解意义。

`SIGINT`：

```text
Interrupt
```

类似终端：

```text
Ctrl+C
```

意思近似：

> 用户希望你停止。

`SIGTERM`：

```text
Terminate
```

意思近似：

> 请正常结束。

程序可以：

```text
保存状态
关闭连接
flush日志
释放资源
```

再退出。

`SIGKILL`：

```text
立即强制终止
```

程序没有机会清理。

所以一个成熟的停止流程通常是：

```text
SIGTERM
 ↓
等待 grace period
 ↓
仍未退出
 ↓
SIGKILL
```

这套逻辑后面在 Docker、Kubernetes 会原样再次出现。

Docker 官方 `docker stop` 就是先给容器主进程发送停止信号，默认使用 `SIGTERM`，等待一段时间后仍未结束再强制终止。citeturn360578search27

---

# 7. 为什么不是直接 SIGKILL

例如 Agent 启动一个 Python 服务：

```python
while True:
    process_task()
```

收到 `SIGTERM`：

```python
def shutdown():
    save_state()
    close_database()
    flush_logs()
```

然后正常退出。

如果直接：

```text
SIGKILL
```

这些清理逻辑都没有机会执行。

后果可能是：

```text
数据库transaction没提交
临时文件未清理
日志没flush
锁文件残留
任务状态不一致
```

所以：

```text
SIGKILL
```

应该理解成：

> 最后的强制手段。

---

# 8. 为什么 Agent 的 timeout 实际上很复杂

假设 Agent Tool：

```text
timeout = 30秒
```

30 秒后 Runtime 需要：

```text
停止任务
```

最简单的实现：

```python
process.kill()
```

可能只杀：

```text
shell
```

但没杀：

```text
child / grandchild
```

于是：

```text
Tool已经timeout
```

但实际程序：

```text
仍然运行
```

这叫：

```text
orphaned process
```

或者更准确地说，原来的父进程退出后，后代进程仍继续存在。

Codex 2026 年公开的 MCP stdio 进程问题里就讨论了类似现象：如果只终止直接 child，而没有正确处理 process group/session，孙进程可能继续持有 pipe；issue 中提出使用独立 session/process group 并整体 kill/wait 的方向。citeturn360578search15

这已经非常接近 Agent Runtime 源码层的调试了。

---

# 9. Process Group

Linux 不只可以：

```text
杀一个 PID
```

还可以把一组进程归入：

```text
process group
```

结构近似：

```text
Process Group 5000
│
├── bash
├── npm
├── node
└── vite
```

然后：

```text
给整个 group 发 signal
```

而不是：

```text
一个一个猜 PID
```

这就是为什么 Unix Agent Runtime 经常会使用：

```text
setsid
setpgid
killpg
```

之类机制。

今天不要求记 API。

只理解：

```text
单PID控制
```

升级成：

```text
一整个执行任务的进程组控制
```

即可。

Codex 的一个 2026 年桌面端 issue 中就能看到其 Git worker 路径存在“启动独立 process group/session，并要求 kill process tree”的实际设计。citeturn360578search22

---

# 10. 为什么后台程序特别容易把 Agent 卡死

例如：

```bash
bash -c "sleep 10 &"
```

Shell：

```text
bash
```

可能马上结束。

但：

```text
sleep
```

仍然运行。

关键问题：

它可能继承：

```text
stdout pipe
stderr pipe
```

于是结构是：

```text
Agent Runtime
  ↑
  │ 等待pipe EOF
  │
 pipe
  ↑
 sleep
```

虽然：

```text
bash已经exit
```

但是：

```text
sleep还持有pipe
```

所以 Runtime：

```text
收不到EOF
```

继续等。

OpenCode 2026 年的公开问题正是这个模式：后台 child 继承 stdout/stderr 后，foreground shell 已退出，但 Tool 仍不会结束；在 Windows 上运行 `vite`、`uvicorn`、`npm run dev` 等长寿命进程也出现过同类行为。citeturn360578search10turn360578search16

---

# 11. 所以 Agent 不能随便用 `&`

Bash：

```bash
npm run dev &
```

看起来像：

```text
放到后台
```

但 Agent Runtime 不一定支持这种用法。

因为：

```text
shell background
```

和：

```text
Agent-managed background task
```

不是一回事。

好的 Agent 系统应该提供：

```text
BackgroundTask Tool
```

或者：

```text
TaskHandle
```

例如：

```text
start_background()
        ↓
task_id = 123
        ↓
Agent继续做其他工作
        ↓
poll(task_id)
        ↓
read logs
        ↓
stop(task_id)
```

而不是简单：

```bash
command &
```

OpenCode 社区也有明确需求：希望运行 `npm test`、`docker build`、`pytest` 等长任务时能以 Agent 可管理的后台任务方式执行，而不是整个 Agent 被阻塞。citeturn360578search23

---

# 12. 企业 Agent Runtime 应该怎么设计

我们后面自己的 Shell Executor 会逐步从：

```python
subprocess.run(command)
```

升级成：

```text
ShellExecutor
 │
 ├── command
 ├── cwd
 ├── env
 ├── timeout
 ├── stdin policy
 ├── stdout capture
 ├── stderr capture
 │
 └── ProcessManager
       │
       ├── PID
       ├── process group
       ├── descendants
       ├── SIGTERM
       ├── grace period
       └── SIGKILL
```

再往上才是：

```text
Agent Tool
 ↓
Approval
 ↓
Sandbox
 ↓
ShellExecutor
```

这就是企业 Agent Runtime 的基本骨架。

---

# 13. PID 1 为什么对 Docker 极其重要

现在提前连接 Docker。

Linux 系统启动后的第一个用户空间进程：

```text
PID 1
```

非常特殊。

普通 Linux：

```text
PID 1
 ↓
systemd/init
 ↓
其他进程
```

而一个 Container 中：

```text
PID 1
```

通常就是你的应用：

```text
PID 1
python app.py
```

或者：

```text
PID 1
node server.js
```

Docker 官方文档明确提醒：容器内 PID 1 在 Linux 中有特殊的信号处理行为，如果应用没有显式处理信号，`SIGINT`/`SIGTERM` 的行为可能和普通进程不同。citeturn360578search8turn360578search21

所以以后会看到：

```text
docker stop agent
```

但容器：

```text
迟迟不退出
```

可能就是：

```text
PID 1没有正确处理SIGTERM
```

---

# 14. PID 1 还有第二个工作：reap child

Unix 子进程退出后，父进程应该：

```text
wait()
```

回收它的退出状态。

如果没人回收，可能出现：

```text
zombie process
```

通常显示：

```text
<defunct>
```

所以容器里一个设计不好的 PID 1：

```text
不会处理signal
+
不会正确reap child
```

对于 Agent workload 很危险。

因为 Agent 特别爱：

```text
大量启动shell
大量启动编译器
大量启动测试
大量启动MCP
大量启动child process
```

所以未来我们的 Agent Container 可能采用：

```text
tini
```

或者正确实现 init/reaping 行为。

今天先记住：

```text
Container PID 1
```

不是普通进程这么简单。

---

# 15. Kimi 的真实 Windows 案例

Kimi CLI 2026 年有一个 PowerShell subprocess bug：

```text
执行任何Shell命令
↓
等待约60秒
↓
The handle is invalid
```

说明同一个 Agent Tool，在：

```text
Linux subprocess
```

和：

```text
Windows PowerShell subprocess
```

底层行为并不一样。citeturn360578search2

这也是为什么企业 Agent Runtime 通常要有：

```text
Unix executor
Windows executor
```

而不是假设：

```text
subprocess is subprocess
```

就可以完全跨平台。

---

# 16. Codex 的真实进程树故障

2026 年 Codex CLI 还出现过 app-server worker 在并行 command burst 后“静默退出”的公开问题：worker 退出后没有把明确 error event 传播给上层，broker 仍在等待 completion，最终 wrapper 看起来就是“卡住”。该 issue 甚至通过追踪 descendant/process tree 来验证 worker 是否消失。citeturn360578search0

这里暴露的是另一个重要原则：

```text
child process died
```

不应该变成：

```text
Agent forever waiting
```

Runtime 应该转化成：

```text
ProcessExitedUnexpectedly
exit_code
signal
last_stdout
last_stderr
```

再返回 Agent。

---

# 17. 今天第一次认识“优雅关闭”

英文：

```text
graceful shutdown
```

理想流程：

```text
用户按Stop
 ↓
Agent Runtime
 ↓
停止发新任务
 ↓
SIGTERM
 ↓
应用停止接受请求
 ↓
处理完当前任务
 ↓
flush logs
 ↓
关闭DB
 ↓
退出child
 ↓
进程退出
```

如果：

```text
10秒仍没退出
```

才：

```text
SIGKILL
```

后面 Kubernetes 会把这个概念变成：

```text
terminationGracePeriodSeconds
```

所以今天实际上已经提前学了 K8s 的核心运行机制。

---

# 18. 今日 Linux/macOS 实验

执行：

```bash
sleep 60 &
```

然后：

```bash
ps -ef | grep sleep
```

找到 PID。

再：

```bash
kill <PID>
```

默认通常发送：

```text
SIGTERM
```

检查：

```bash
ps -ef | grep sleep
```

然后重新：

```bash
sleep 60 &
```

找到 PID：

```bash
kill -9 <PID>
```

这次是：

```text
SIGKILL
```

你今天只需要感受：

```text
kill
```

和：

```text
kill -9
```

不是同一层级。

再试：

```bash
bash -c 'sleep 30 & echo shell-done'
```

观察：

```text
shell-done
```

会马上出来，但：

```bash
ps -ef | grep sleep
```

还能找到子进程。

这就是今天最关键的实验。

---

# Windows PowerShell 实验

启动：

```powershell
$p = Start-Process powershell `
  -ArgumentList "-Command Start-Sleep 60" `
  -PassThru
```

查看：

```powershell
$p.Id
```

然后：

```powershell
Get-Process -Id $p.Id
```

停止：

```powershell
Stop-Process -Id $p.Id
```

再观察：

```powershell
Get-Process -Id $p.Id
```

这就是：

```text
Process handle / PID
```

的最基础体验。

---

# 今日企业故障案例

现象：

```text
Agent执行：
npm run dev

点击Cancel

UI显示：
Cancelled

但是下一次：
npm run dev

报：
Port 5173 already in use
```

不要首先判断：

```text
Vite坏了
```

正确故障树是：

```text
Cancel
 ↓
Shell父进程是否退出？
 ↓
node child是否还存在？
 ↓
process group是否被kill？
 ↓
stdout/stderr pipe是否还被child持有？
 ↓
端口由哪个PID占用？
```

Linux：

```bash
ps -ef | grep node
```

或者：

```bash
lsof -i :5173
```

Windows：

```powershell
Get-NetTCPConnection -LocalPort 5173
```

再：

```powershell
Get-Process -Id <PID>
```

如果发现：

```text
Agent取消了
但是node还活着
```

那么：

```text
模型没有问题
```

问题在：

```text
Process Manager
```

---

## 和 Codex / OpenCode / Kimi 的源码思路对比

这一课不需要把三个项目完整源码读完，但已经可以理解它们为什么都需要复杂的 shell/process 管理。OpenCode 当前公开 issue 明确暴露了 background child + pipe 的问题；Kimi CLI 有 detached child/PowerShell subprocess 问题；Codex 则有 process-tree termination、MCP child process 和 worker lifecycle 类问题。citeturn360578search10turn360578search4turn360578search15

Claude Code 核心产品不按开源源码分析，但从 Agent SDK / Bash Tool 的行为模型来看，同样必须解决：

```text
process creation
timeout
output capture
permission
termination
```

这些 Runtime 问题。

所以最终不同 Coding Agent 的 UI 差异很大，但底层都会碰到：

```text
OS process model
```

---

## 今天对应的岗位能力

今天已经进入 **Agent Runtime / DevOps / Container Runtime / SRE** 的交叉区域：

`PID、PPID、process tree、process group、SIGINT、SIGTERM、SIGKILL、graceful shutdown、background process、orphan process、PID 1`。

其中最重要的不是命令，而是形成这个判断：

```text
Agent卡住
≠
LLM还在思考

可能是：
process没退出
child没退出
pipe没关闭
signal没送达
Runtime没传播异常
```

这会极大提升你对 Codex、Claude Code、OpenCode、Kimi Code 的使用信心，因为以后看到它“卡”，你已经有能力区分 **模型层故障** 和 **运行时层故障**。

### 今日检查题

1. 为什么只 kill Shell 的 PID，`npm run dev` 启动的 Node 进程仍可能存在？
2. `SIGTERM` 和 `SIGKILL` 最大区别是什么？
3. 为什么后台 child 进程会导致 Agent Tool 等不到结束？
4. 为什么 Docker 里的 PID 1 值得特殊关注？
5. Agent 点击 Cancel 后端口仍被占用，你首先应该检查模型还是 process tree？

参考答案核心是：**进程可以形成父子树；SIGTERM允许程序处理并优雅退出而SIGKILL强制终止；child可能继续持有 stdout/stderr pipe；容器 PID 1 有特殊信号/子进程管理责任；端口仍占用应先排查进程树和 Runtime。**

下一课进入 **Linux 文件系统、用户、权限、`rwx`、`chmod`、`chown` 与 Agent Sandbox**：为什么 Agent 明明“看得到文件”，却可能读不了、写不了，以及 Docker/Kubernetes 的 `root`、非 root 用户和最小权限到底解决什么问题。