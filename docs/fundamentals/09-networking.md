# 第 9 课：HTTP / REST / SSE / WebSocket——把 Agent 的“长任务”和“实时输出”拆开

今天解决企业 Agent 平台里一个非常实际的问题：

> Agent 一个任务可能运行 10 分钟，但用户希望提交任务后马上拿到 `task_id`，同时又能实时看到“正在读文件、正在运行测试、正在调用模型”。

正确思路不是让一个 HTTP 请求挂 10 分钟，而是把 **任务生命周期** 和 **事件流** 分成两条通道。

```text
POST /tasks
     │
     ▼
API ───────→ Redis Queue ─────→ Agent Runner
 │                               │
 │ 返回 task_id                  ├─ git
 │                               ├─ LLM
 ▼                               ├─ shell
202 Accepted                    ├─ test
                                 │
GET /tasks/{id}/events           │
     │                           │
     └──────── SSE ◀─────────────┘
```

这基本就是我们上一课 `API + Redis + Runner + PostgreSQL` 架构继续往前走。

---

## 1. 为什么 Agent 不适合普通同步 HTTP

传统接口可能是：

```http
POST /calculate
```

几十毫秒后：

```http
200 OK
```

但 Coding Agent 的调用链可能是：

```text
HTTP
 ↓
LLM
 ↓
rg搜索代码
 ↓
读取文件
 ↓
LLM
 ↓
修改代码
 ↓
dotnet test
 ↓
测试失败
 ↓
LLM
 ↓
继续修改
 ↓
git diff
 ↓
最终结果
```

因此执行时间可能从几秒延伸到数分钟甚至更久。

企业 API 更合理的是：

```http
POST /tasks
```

立即得到：

```json
{
  "id": "task-101",
  "status": "queued"
}
```

HTTP 状态可以设计成：

```text
202 Accepted
```

表示：

> 请求已经接受，但工作尚未完成。

真正执行：

```text
API
 ↓
Queue
 ↓
Runner
```

所以 API Server 不需要一直占着一个普通请求等待 Agent 完成。

---

# 2. REST 在 Agent 平台里负责什么

可以设计：

```text
POST   /tasks
GET    /tasks/{id}
DELETE /tasks/{id}

GET    /tasks/{id}/events
```

例如：

```http
POST /tasks
Content-Type: application/json
```

请求：

```json
{
  "repository": "sample-app",
  "prompt": "修复 HTTP timeout 并补充测试"
}
```

返回：

```json
{
  "task_id": "task-101",
  "status": "queued"
}
```

然后：

```http
GET /tasks/task-101
```

得到：

```json
{
  "task_id": "task-101",
  "status": "running",
  "created_at": "...",
  "started_at": "..."
}
```

最终：

```json
{
  "task_id": "task-101",
  "status": "succeeded",
  "result": "..."
}
```

这条 REST API 管的是：

```text
Task Resource
```

不是负责逐 Token 输出。

---

# 3. 但用户还想实时看 Agent 干什么

例如 Codex 的 UI 会不断出现：

```text
Searching files...

Reading src/App.cs...

Running:
dotnet test

2 tests failed...

Editing HttpClient.cs...
```

如果只能：

```http
GET /tasks/task-101
```

不断轮询：

```text
running
running
running
running
running
succeeded
```

体验很差。

所以我们需要：

```text
Event Stream
```

例如：

```text
task.started

agent.message.delta

tool.started

tool.output

tool.completed

test.failed

file.changed

agent.message.delta

task.completed
```

现在就到了 SSE。

---

# 4. SSE 是什么

SSE：

```text
Server-Sent Events
```

核心方向是：

```text
Server
   │
   │ event
   │ event
   │ event
   ▼
Client
```

浏览器的 `EventSource` 接口就是为服务器持续向客户端推送事件设计的；它是**单向的 Server → Client**。citeturn408311search3turn408311search14

这非常适合：

```text
Agent → Web UI
```

因为 Agent 的大量实时信息本来就是：

```text
后端不断告诉前端：
“发生什么了”
```

---

# 5. SSE 数据长什么样

一个 SSE endpoint 通常返回：

```http
Content-Type: text/event-stream
```

数据：

```text
event: task.started
data: {"task_id":"task-101"}

event: tool.started
data: {"tool":"shell","command":"dotnet test"}

event: tool.output
data: {"text":"Build started..."}

event: tool.completed
data: {"exit_code":1}

event: task.completed
data: {"status":"failed"}
```

SSE 的事件流本质仍建立在 HTTP 连接之上；浏览器通过 `EventSource` 持续接收服务端事件。citeturn408311search7

所以：

```text
HTTP
```

和：

```text
Streaming
```

并不是矛盾关系。

可以是：

```text
HTTP + 长连接 + event stream
```

---

# 6. OpenAI 当前就是这样 Streaming

截至 **2026 年 8 月 20 日**，OpenAI Responses API 官方文档仍明确使用：

```text
stream=true
```

通过 **Server-Sent Events（SSE）** 流式返回 Responses API 结果；Responses 流不是单纯字符串，而是 typed events。citeturn971393search0turn971393search5

例如事件可能包括：

```text
response.created
response.output_text.delta
response.completed
error
``` citeturn971393search5


这非常重要。

错误设计：

```text
Agent Stream
 ↓
永远只是文本字符串
```

更好的设计：

```text
Agent Stream
 ↓
Typed Events
```

例如：

```json
{
  "type": "tool.started",
  "tool": "shell",
  "command": "dotnet test"
}
```

因为前端看到：

```text
tool.started
```

可以显示：

```text
🔧 Running dotnet test
```

而不是把所有东西塞到聊天文字中。

---

# 7. Anthropic 也是事件流

Claude Messages API 设置：

```json
{
  "stream": true
}
```

同样通过 SSE 增量返回内容，而且 Anthropic 的 streaming 不只覆盖文本，也覆盖 tool use 等事件。citeturn971393search1turn971393search2

这进一步说明：

```text
LLM streaming
```

实际应该理解成：

```text
event stream
```

而不只是：

```text
token
token
token
```

例如可能出现：

```text
message_start
content_block_start
content_block_delta
content_block_stop
message_delta
message_stop
```

于是 Agent Runtime 可以：

```text
LLM Event
     ↓
转换
     ↓
Platform Event
     ↓
Web UI
```

---

# 8. Tool Call 甚至也能 Streaming

假设模型准备调用：

```json
{
  "command":
    "dotnet test src/sample-app.Tests/sample-app.Tests.csproj"
}
```

传统模式可能要等整个 JSON 完成：

```text
模型生成完整tool arguments
 ↓
Runtime解析JSON
 ↓
执行
```

现在一些模型 API 可以增量传输 tool 参数。

OpenAI 官方 function-calling 文档支持 streaming function calls，也就是调用参数生成过程中就可看到增量事件。citeturn971393search17

Anthropic 当前甚至提供 fine-grained tool streaming，让工具输入参数在生成过程中直接流出，以降低大参数调用的首片段延迟；官方也提醒这种情况下中间数据可能还是部分或无效 JSON，因此消费者必须正确累积和解析。citeturn971393search6

这里出现一个很重要的工程边界：

```text
streamed argument
≠
valid final argument
```

不要看到：

```text
{"command":"rm
```

就开始执行。

必须等到：

```text
完整、验证通过的Tool Call
```

再进入：

```text
Approval
 ↓
Sandbox
 ↓
Execution
```

---

# 9. Agent Stream 不能直接等于 LLM Stream

这是今天最重要的架构点之一。

LLM 只知道：

```text
response.output_text.delta
tool_call...
```

但企业 Agent 还有很多 LLM 不知道的事件：

```text
queue.received
workspace.created
git.checkout.completed
container.started
sandbox.denied
tool.started
tool.output
tool.completed
test.failed
retry.scheduled
approval.required
task.completed
```

所以应该有：

```text
Provider Event
      ↓
Agent Runtime
      ↓
统一事件模型
      ↓
Platform Event Bus
      ↓
Web/API
```

不能设计成：

```text
OpenAI event
直接暴露给Web

Claude event
换另一个格式

Kimi event
又一个格式
```

否则以后切模型：

```text
前端全部要改
```

---

# 10. 我们自己的统一事件模型

第一版可以定义：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class AgentEvent:
    task_id: str
    type: str
    data: dict[str, Any]
```

例如：

```python
AgentEvent(
    task_id="task-101",
    type="tool.started",
    data={
        "tool": "shell",
        "command": "dotnet test",
    },
)
```

或者：

```python
AgentEvent(
    task_id="task-101",
    type="tool.completed",
    data={
        "exit_code": 1,
        "duration_ms": 4250,
    },
)
```

最终前端永远消费：

```text
AgentEvent
```

不直接依赖：

```text
OpenAIEvent
AnthropicEvent
KimiEvent
```

这就是：

```text
Adapter Layer
```

---

# 11. 和 C# 很像

C# 可以设计：

```csharp
public record AgentEvent(
    string TaskId,
    string Type,
    object Data
);
```

再：

```csharp
public interface IAgentProvider
{
    IAsyncEnumerable<AgentEvent> RunAsync(
        AgentRequest request,
        CancellationToken ct);
}
```

于是：

```text
OpenAIProvider
ClaudeProvider
KimiProvider
LocalModelProvider
```

都实现：

```text
IAgentProvider
```

这个设计以后会直接进入：

```text
LLM Routing
```

---

# 12. FastAPI 做一个最小 SSE

例如：

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio
import json

app = FastAPI()


async def event_stream(task_id: str):

    events = [
        ("task.started", {"task_id": task_id}),
        (
            "tool.started",
            {
                "tool": "shell",
                "command": "dotnet test",
            },
        ),
        (
            "tool.completed",
            {
                "exit_code": 0,
            },
        ),
        (
            "task.completed",
            {
                "status": "succeeded",
            },
        ),
    ]

    for event_type, data in events:

        yield (
            f"event: {event_type}\n"
            f"data: {json.dumps(data)}\n\n"
        )

        await asyncio.sleep(1)


@app.get("/tasks/{task_id}/events")
async def events(task_id: str):

    return StreamingResponse(
        event_stream(task_id),
        media_type="text/event-stream",
    )
```

然后浏览器：

```javascript
const stream =
    new EventSource(
        "/tasks/task-101/events"
    );

stream.addEventListener(
    "tool.started",
    event => {

        console.log(
            JSON.parse(event.data)
        );
    }
);
```

这已经是一个真正的：

```text
Agent Web Streaming
```

最小原型。

---

# 13. SSE 和 WebSocket 到底怎么选

先建立简单判断：

```text
SSE

Server ─────────→ Client
```

WebSocket：

```text
Client ⇄ Server
```

MDN 也明确区分了 EventSource 的单向 Server → Client 和 WebSocket 的双向通信能力。citeturn408311search14turn408311search33

所以普通 Coding Agent Web UI：

```text
POST /task
+
服务端不断发送progress
```

SSE 往往已经够用。

如果需要：

```text
用户不断实时发控制事件
模型不断回事件

voice/audio
interactive terminal
实时双向输入
```

WebSocket 会更自然。

---

# 14. OpenAI 为什么两种都存在

OpenAI 当前文档本身就很好地展示了这两种用途：

```text
普通Responses streaming
→ SSE
```

而需要持久双向会话时，Responses / Realtime 有 WebSocket 模式；Realtime WebSocket 明确通过客户端事件和服务端事件共同管理 session。citeturn971393search0turn971393search9turn971393search15

所以不要形成：

```text
WebSocket比较高级
所以都用WebSocket
```

的错误认识。

工程选择应该基于通信模式。

---

# 15. Coding Agent 为什么常常 SSE 就够

假设网页：

```text
用户：
“修复bug”
```

只需：

```http
POST /tasks
```

用户后面绝大多数时间：

```text
看Agent输出
```

即：

```text
Server → Browser
```

真正的用户操作：

```text
Cancel
Approve
Reject
Send message
```

完全可以再使用普通 REST：

```text
POST /tasks/101/cancel

POST /tasks/101/approvals/22
```

于是结构很清楚：

```text
Command:
HTTP POST

Events:
SSE
```

这是非常好用的企业设计。

---

# 16. Command 和 Event 要分开

比如：

```text
用户点击Cancel
```

这是：

```text
Command
```

因为用户要求系统改变状态。

可以：

```http
POST /tasks/task-101/cancel
```

而：

```text
task.cancelled
```

是：

```text
Event
```

代表：

> 系统中某件事情已经发生。

因此：

```text
Command:
CancelTask

Event:
TaskCancelled
```

这个区分以后学习：

```text
Event Driven Architecture
CQRS
Kafka
```

都会再次遇到。

---

# 17. 为什么不能把 Stream 当数据库

用户浏览器断网：

```text
SSE断开
```

如果所有状态只存在：

```text
SSE stream
```

用户回来以后：

```text
不知道Agent做过什么
```

因此：

```text
Streaming
```

应该用于：

```text
实时传递
```

而：

```text
PostgreSQL / Event Store
```

负责：

```text
持久状态
```

理想结构：

```text
Agent Runner
   │
   ├── Event → Redis/Bus → SSE
   │
   └── State → PostgreSQL
```

比如用户断线回来：

```http
GET /tasks/task-101
```

仍然能获得权威状态。

Anthropic 当前 Managed Agents 的 event-stream 文档也专门区分了增量 preview 与最终 authoritative buffered agent message：增量预览方便实时显示，但最终记录才应该作为权威结果。citeturn971393search10

这个原则非常值得我们采用。

---

# 18. Token Stream 也不是最终记录

假设模型流：

```text
Hel
Hello
Hello wor
Hello world
```

这些：

```text
delta
```

不是四条消息。

最终只是：

```text
Hello world
```

所以数据库绝不能：

```text
每收到一个token
就当一条assistant message
```

更合理：

```text
delta
↓
实时UI

completed message
↓
持久化
```

否则：

```text
数据库爆炸
消息语义混乱
恢复困难
```

---

# 19. 为什么 Agent 事件比 Token 更重要

Coding Agent 最有价值的实时信息，很多根本不是 token：

```text
workspace.created

git.branch.created

tool.started:
rg "IOutputFormatter"

tool.completed:
37 matches

file.read:
Formatter.cs

patch.applied:
Formatter.cs

test.started

test.failed:
2 failures

retry.started

test.passed

task.completed
```

这些事件可以直接用于：

```text
UI
Trace
Audit
Metrics
Debugging
Eval
```

于是你会发现：

> 设计 Agent Event Schema 本身就是 Agent 平台工程的重要工作。

---

# 20. Stream 还必须考虑 Backpressure

假设 Agent 执行：

```bash
npm test
```

1 秒产生：

```text
10 MB日志
```

但浏览器只能消费：

```text
100 KB/s
```

怎么办？

不能无限：

```text
内存缓存
```

否则 API Server 迟早 OOM。

这个现象叫：

```text
backpressure
```

也就是：

> Producer 比 Consumer 快。

企业 Agent 可以：

```text
截断日志
聚合日志
按行batch
限制单事件大小
只保留tail
写对象存储
UI只显示最近N KB
```

以后讲：

```text
Kafka
OpenTelemetry
logging
```

还会深入。

---

# 21. Stream 也存在安全问题

假设：

```bash
echo $OPENAI_API_KEY
```

错误地被 Agent 执行。

stdout：

```text
sk-xxxx
```

如果系统直接：

```text
stdout
 ↓
SSE
 ↓
浏览器
```

密钥就泄漏了。

因此：

```text
Tool Output
```

不能简单原样广播。

至少应该经过：

```text
Tool Output
 ↓
Redaction
 ↓
Size Limit
 ↓
Audit Policy
 ↓
Event
```

OpenAI 官方在涉及 MCP / Deep Research 的生产建议中也强调应记录并审查向 MCP 等外部工具传递的数据轨迹，以确认数据共享符合预期。citeturn971393search19

所以：

```text
Streaming
```

不是绕过安全策略的通道。

---

# 22. 今天把 Mini Agent Platform 再升级一层

现在架构变成：

```text
                    Browser
                       │
           ┌───────────┴──────────┐
           │                      │
      POST /tasks          GET /events
           │                      ▲
           ▼                      │ SSE
         API                  Event Gateway
           │                      ▲
           ▼                      │
       Redis Queue             Event Bus
           │                      ▲
           ▼                      │
       Agent Runner ──────────────┘
           │
       ┌───┼────┐
       ▼   ▼    ▼
      LLM Git  Shell
           │
           ▼
      PostgreSQL
```

现在已经不是：

```text
“写一个Agent脚本”
```

而是在形成：

```text
Agent Platform
```

---

# 23. 今日 10～15 分钟实验

继续上一课的 FastAPI 项目。

新增：

```text
GET /tasks/{id}/events
```

然后先不要真正连接 LLM。

人为生成：

```text
task.started
tool.started
tool.output
tool.completed
task.completed
```

五类事件。

浏览器或：

```bash
curl -N \
  http://localhost:8000/tasks/task-101/events
```

观察事件逐个出现。

`-N` 的目的，是减少 curl 对输出的缓冲，让你更明显地看到流。

期望：

```text
event: task.started
data: ...

event: tool.started
data: ...

event: tool.completed
data: ...
```

如果看到不是一次性全部出现，而是陆续出现，这一课的最关键实验就完成了。

---

# 企业故障排查案例

现象：

```text
Agent后台正常执行

数据库：
running

日志：
也一直有输出

但是网页：
几分钟以后一次性出现所有内容
```

不要先怀疑：

```text
模型没有stream
```

故障树应该是：

```text
LLM真的在stream吗？
        ↓
Runner是否边收边发？
        ↓
Event Bus是否实时？
        ↓
API是否flush？
        ↓
反向代理是否buffer？
        ↓
浏览器是否实时消费？
```

尤其以后加入：

```text
Nginx
Ingress
Cloud Load Balancer
```

之后，Proxy buffering 就会成为重要排查方向。

也就是说：

```text
模型streaming
≠
用户一定看到streaming
```

中间还有：

```text
Model
↓
SDK
↓
Runner
↓
Queue
↓
API
↓
Proxy
↓
Browser
```

任何一层都可能缓冲。

---

# Codex / Claude / Kimi 对今天课程的意义

OpenAI 当前 Responses API 已经明确采用 typed SSE event 模型；Anthropic Messages API 也以 SSE 方式增量提供 text、tool use 等事件。citeturn971393search0turn971393search1

Kimi Code 当前官方仓库则继续定位为能读写文件、执行 shell、搜索网页并依据反馈自主决定下一步的终端 Coding Agent，因此如果把这种 CLI Runtime 服务化，同样需要把其 CLI/TUI 事件转换成稳定的平台事件接口。citeturn408311search25

这里要继续坚持：

**我们可以分析 Kimi 公开源码和协议；Claude Code 未公开的产品核心不能伪装成源码。API streaming 行为则可以依据 Anthropic 官方文档准确分析。**

---

## 岗位能力映射

今天真正训练的是：

**HTTP、REST、异步 Job API、202 Accepted、SSE、WebSocket、Event Schema、Streaming、Adapter、Backpressure、状态持久化、Agent Web Gateway 和流式安全。**

企业面试如果问：

> Coding Agent 的任务可能执行 30 分钟，Web 前端怎样实时显示进度？

你现在应该可以回答：

> 用异步任务模型先通过 REST 创建 Task 并立即返回 `task_id`，Agent Runner 在后台消费任务；运行事件通过统一事件模型进入 Event Bus，Web 端通过 SSE 等方式订阅。最终状态和权威结果持久化到数据库，Streaming 只负责实时传递而不能替代持久状态。需要双向高频实时通信时再考虑 WebSocket。

这个答案已经比：

> “用 WebSocket。”

完整得多。

### 今日检查题

1. 为什么 Agent 长任务不适合让普通 HTTP 请求一直阻塞等待？
2. SSE 与 WebSocket 最大的通信方向差异是什么？
3. 为什么平台不能直接把 OpenAI/Claude 原始 streaming event 暴露给业务前端？
4. 为什么 token delta 不应该直接作为最终数据库消息？
5. LLM 已经开启 Streaming，为什么浏览器仍可能几分钟后一次性看到结果？

答案核心是：**Agent 应作为异步 Job；SSE 主要 Server→Client、WebSocket 双向；需要统一 Provider-independent Event Schema；delta 是增量显示而非权威最终状态；SDK、Event Bus、API、反向代理、浏览器等任一层都可能发生缓冲。**

**下一课进入 Nginx / Reverse Proxy / DNS / TLS / HTTPS**：实际把今天的 SSE API 放到反向代理后面，并解释为什么生产环境不是直接把 `FastAPI:8000` 暴露给互联网，以及为什么 Agent Streaming 最容易在 Nginx、Ingress 和负载均衡层出现“本地正常、上线不流式”的问题。