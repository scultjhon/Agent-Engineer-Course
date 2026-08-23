# 第 8 课：Docker Compose + Agent 服务化——为什么企业 Agent 平台不能把所有东西塞进一个进程

今天从“单个 Agent Runner 容器”升级为一个最小企业服务栈：

```text
API
↓
任务队列
↓
Agent Runner
↓
PostgreSQL / Redis
```

目标是理解一个很核心的架构问题：

> 为什么企业 Agent 系统不能只是 `python agent.py` 一直跑，而要拆成 API、Runner、队列、数据库等多个服务？

Docker Compose 正适合用来完成这一阶段。Docker 官方把 Compose 定义为：用一个 `compose.yaml` 描述和运行多容器应用，包括 services、networks、volumes 等。citeturn965719search8turn965719search33

---

## 1. 先看最小企业 Agent 架构

假设用户提交：

> 修复 Git 仓库里的 timeout bug。

不要直接：

```text
HTTP请求
 ↓
Agent开始跑30分钟
 ↓
HTTP一直等
```

更好的结构：

```text
用户
 ↓ HTTP
API Service
 ↓
创建 Task
 ↓
Redis Queue
 ↓
Agent Runner
 ↓
Git / Shell / LLM
 ↓
结果
 ↓
PostgreSQL
```

API 只负责：

```text
接收任务
鉴权
校验参数
创建task_id
查询状态
```

Runner 负责：

```text
克隆/挂载仓库
构建context
调用模型
执行tool
测试
保存结果
```

数据库负责：

```text
task状态
Agent执行历史
用户
审计记录
token/cost
结果
```

队列负责：

```text
“还有哪些任务等待执行？”
```

这就是第一次真正接触：

**control plane 与 worker plane 分离。**

可以暂时理解为：

```text
API / Scheduler
= 控制任务

Runner
= 真正干活
```

---

# 2. 为什么不能让 API 进程直接跑 Agent

例如：

```python
@app.post("/agent")
def run_agent():
    subprocess.run(...)
```

问题很多。

如果 Agent：

```text
运行25分钟
```

HTTP 请求可能：

```text
timeout
```

如果 Agent：

```text
npm run dev
```

卡住，Web API worker 也可能被拖住。

如果一次来了：

```text
100个请求
```

就可能变成：

```text
100个Agent
+
100组shell
+
100份LLM调用
```

服务器迅速失控。

所以：

```text
request handling
```

和：

```text
long-running Agent workload
```

通常应该分开。

后面 Kubernetes 会把 Runner 进一步升级成：

```text
一个Task
↓
一个Pod/Job
```

---

# 3. Compose 的 `service`

先写第一版：

```yaml
services:

  api:
    build: ./api

  runner:
    build: ./runner

  redis:
    image: redis:8

  postgres:
    image: postgres:18
```

这里每一个：

```text
api
runner
redis
postgres
```

都是一个 Compose service。

Docker Compose 会依据这个配置创建和管理多个容器，并自动处理默认网络等基础设施。citeturn965719search13turn965719search31

运行：

```bash
docker compose up
```

可能得到：

```text
agent-api-1
agent-runner-1
agent-redis-1
agent-postgres-1
```

这就比上一课：

```bash
docker run ...
```

手动管理多个 container 更适合完整应用。

---

# 4. Compose 自动创建网络

这是今天非常重要的一点。

假设：

```yaml
services:

  api:
    ...

  postgres:
    image: postgres:18
```

Compose 会给应用创建网络。

于是 API 中通常不应该：

```text
DATABASE_HOST=localhost
```

而应该：

```text
DATABASE_HOST=postgres
```

为什么？

因为：

```text
api container
```

里的：

```text
localhost
```

指的是：

```text
api container自己
```

不是 PostgreSQL container。

真正结构：

```text
Compose Network

┌──────────────┐
│ api          │
│              │
│ localhost    │ ← api自己
└──────┬───────┘
       │
       │ postgres:5432
       ↓
┌──────────────┐
│ postgres     │
└──────────────┘
```

Compose 提供的服务名可以作为网络内 DNS 名称，所以应用可以通过：

```text
postgres:5432
redis:6379
```

通信。Docker 官方 PostgreSQL Compose 指南也明确使用 Compose 自动网络实现服务间通信。citeturn965719search31

这是 Docker 初学最常见的问题之一：

> 宿主机访问数据库用 `localhost:5432`，为什么容器里 `localhost:5432` 不通？

原因就是：

```text
localhost是“当前网络命名空间中的自己”
```

---

# 5. 第一次真正理解 Port

我们可以写：

```yaml
postgres:
  image: postgres:18
  ports:
    - "5432:5432"
```

意思近似：

```text
Host 5432
   ↓
Container 5432
```

所以你的 Mac/Windows：

```text
localhost:5432
```

可以访问 PostgreSQL。

但如果：

```text
只有api需要访问postgres
```

其实生产环境未必需要把 PostgreSQL 暴露到 host。

因为：

```text
api
 ↓
Compose internal network
 ↓
postgres:5432
```

已经够了。

也就是说：

> **能不暴露的端口，不必全部暴露。**

这是最小攻击面的一个实际体现。

---

# 6. `environment`

PostgreSQL container 通常需要：

```yaml
postgres:
  image: postgres:18

  environment:
    POSTGRES_DB: agentdb
    POSTGRES_USER: agent
    POSTGRES_PASSWORD: devpassword
```

API：

```yaml
api:
  environment:
    DATABASE_URL: postgresql://agent:devpassword@postgres:5432/agentdb
```

这里再次用到你已经学过的：

```text
environment variable
```

Container 启动：

```text
PID 1
 ↓
继承 environment
 ↓
应用读取 DATABASE_URL
```

例如 Python：

```python
import os

db = os.environ["DATABASE_URL"]
```

C# 类比：

```csharp
Environment.GetEnvironmentVariable("DATABASE_URL");
```

但是注意：

```text
environment variable
≠
完善的Secret管理系统
```

开发环境暂时可以这样做。

生产密码、API Key 后面会转向：

```text
Docker secrets
Kubernetes Secret
Vault / Cloud Secret Manager
```

---

# 7. Volume：数据库不能随着 container 删除

上一课说过：

```text
Container writable layer
```

通常和 container 生命周期绑定。

如果 PostgreSQL 数据也只放那里：

```bash
docker compose down
```

删除 container 后，数据可能丢失。

因此：

```yaml
postgres:

  volumes:
    - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

结构：

```text
postgres container
       │
       ▼
/var/lib/postgresql/data
       │
       ▼
Docker managed volume
       │
       └── postgres_data
```

Docker 官方把 volume 作为 Docker 管理的持久化存储，并指出其生命周期可独立于具体 container。citeturn965719search8

所以：

```text
container
```

可以：

```text
destroy / recreate
```

而数据还在。

这是云原生里一个很重要的思维：

> **进程/容器可以是短暂的，状态需要单独管理。**

---

# 8. 为什么 Agent Runner 反而应该尽量无状态

假设：

```text
Runner A
```

执行任务 101。

如果所有任务状态只保存在 Runner 内存：

```python
tasks = {}
```

Runner crash：

```text
任务状态一起没了
```

因此更合理：

```text
Runner
↓
只负责execution

Task state
↓
PostgreSQL

Queue
↓
Redis
```

那么：

```text
Runner A crash
```

可以：

```text
Runner B
```

重新接手。

这叫：

```text
stateless worker
```

以后 Kubernetes：

```text
Deployment replicas=5
```

就是建立在类似理念之上。

---

# 9. Redis 在这里做什么

Redis 很多人只理解成：

```text
缓存
```

其实它也可以作为：

```text
queue / stream
```

例如 Redis Streams 是 append-only log，并支持 consumer group，多 consumer 可以协作消费消息。citeturn965719search4turn965719search11

简化理解：

```text
API
 │
 │ 新任务
 ▼
Redis Stream
 │
 ├── task101
 ├── task102
 └── task103
       │
       ▼
Consumer Group: agent-runners
       │
   ┌───┴────┐
   ↓        ↓
Runner A  Runner B
```

Redis consumer group 可以让多个 worker 像一个团队一样分摊消息；`XREADGROUP` 用来消费，而 `XACK` 表示任务已处理确认。citeturn965719search34turn965719search11

这已经开始接近：

```text
Agent Scheduler
```

---

# 10. 为什么要有 ACK

假设：

```text
Runner A
```

取得：

```text
task101
```

然后执行到一半：

```text
crash
```

如果队列已经直接把 task101 删除：

```text
任务丢失
```

更好的队列设计是：

```text
任务分配
 ↓
pending
 ↓
Runner处理
 ↓
成功
 ↓
ACK
```

如果：

```text
Runner挂了
```

系统可以发现：

```text
pending但很久没ACK
```

然后：

```text
retry
```

Redis Streams 的 consumer group 就维护 pending entries，可用于追踪已投递但尚未确认的消息。citeturn965719search34

这直接关系到 Agent 后面最重要的问题之一：

```text
Agent任务到底执行了一次，
还是可能执行两次？
```

稍后我们会进入：

```text
at-least-once
idempotency
retry
```

---

# 11. PostgreSQL 为什么不直接被 Redis 替代

可以先这么分：

### Redis

适合：

```text
任务排队
临时状态
锁
缓存
事件流
```

### PostgreSQL

适合：

```text
Task记录
User
Repository
Audit
Agent run历史
Token cost
最终结果
```

PostgreSQL 最大价值之一是：

```text
transaction
```

数据库事务可以把多个操作看成一个整体：全部完成或全部失败。citeturn965719search7

例如创建 Agent Task：

```text
插入task
+
写audit log
+
扣quota
```

可以放入 transaction。

如果中间失败：

```text
rollback
```

避免：

```text
quota扣了
但task没创建
```

这就是数据库工程开始进入 Agent 平台。

---

# 12. 第一个 `compose.yaml`

这一周的小项目可以升级成：

```text
mini-agent-platform/
│
├── compose.yaml
├── api/
│   ├── Dockerfile
│   └── app.py
│
└── runner/
    ├── Dockerfile
    └── runner.py
```

先写：

```yaml
services:

  redis:
    image: redis:8-alpine

  postgres:
    image: postgres:18-alpine

    environment:
      POSTGRES_DB: agentdb
      POSTGRES_USER: agent
      POSTGRES_PASSWORD: agent-dev

    volumes:
      - postgres_data:/var/lib/postgresql/data

  api:
    build: ./api

    environment:
      REDIS_HOST: redis
      DATABASE_HOST: postgres

    depends_on:
      - redis
      - postgres

  runner:
    build: ./runner

    environment:
      REDIS_HOST: redis
      DATABASE_HOST: postgres

    depends_on:
      - redis
      - postgres

volumes:

  postgres_data:
```

执行：

```bash
docker compose up --build
```

现在有：

```text
API
Runner
Redis
PostgreSQL
```

一起启动。

---

# 13. 但 `depends_on` 有一个大坑

很多初学者看到：

```yaml
depends_on:
  - postgres
```

以为：

> PostgreSQL 已经准备好接受连接了。

不一定。

Docker 官方明确区分：

```text
container已经启动
```

和：

```text
service已经ready
```

`depends_on` 可以控制启动顺序，但真正等待服务健康通常应该结合 `healthcheck` 和 `condition: service_healthy`。citeturn965719search2turn965719search20

例如：

```text
postgres container started
↓
PostgreSQL还在初始化数据库
↓
API马上连接
↓
Connection refused
```

于是出现经典：

```text
“明明 depends_on 了为什么数据库还是连不上？”
```

因为：

```text
started ≠ ready
```

---

# 14. Healthcheck

可以写：

```yaml
postgres:

  image: postgres:18-alpine

  healthcheck:
    test:
      [
        "CMD-SHELL",
        "pg_isready -U agent -d agentdb"
      ]

    interval: 5s
    timeout: 3s
    retries: 10
```

然后：

```yaml
api:

  depends_on:

    postgres:
      condition: service_healthy
```

结构：

```text
postgres container start
       ↓
healthcheck
       ↓
not healthy
       ↓
继续检查
       ↓
healthy
       ↓
API start
```

Docker 官方 Compose 指南也直接使用 healthcheck + `depends_on` 来消除服务启动 race condition。citeturn965719search20

这个概念后面会直接升级成 Kubernetes：

```text
readinessProbe
livenessProbe
startupProbe
```

所以现在一定要理解。

---

# 15. `started ≠ healthy ≠ useful`

再深入一点。

有三层：

```text
Process started
```

表示：

```text
PID存在
```

第二层：

```text
Service healthy
```

例如：

```text
PostgreSQL接受连接
```

第三层：

```text
Application ready
```

例如：

```text
migration已经完成
required table存在
模型服务加载完成
```

对 Agent 模型服务尤其明显。

例如 Hugging Face inference server：

```text
container已经启动
```

但还在：

```text
加载30GB模型权重
```

此时：

```text
PID存在
```

不代表：

```text
可以推理
```

所以以后 Agent 部署：

```text
Process
Container
Service
Model
```

每一层都可能有自己的 ready 条件。

---

# 16. Agent Runner 应该怎样拿任务

现在写最简单伪代码：

```python
while True:

    task = queue.pop()

    if task is None:
        sleep()

    else:
        execute_agent(task)
```

但企业版会慢慢变成：

```python
while True:

    task = queue.claim()

    try:

        mark_running(task)

        result = run_agent(task)

        save_result(result)

        queue.ack(task)

    except Exception as e:

        mark_failed(task, e)

        retry_or_dead_letter(task)
```

这里已经出现几个以后必须掌握的词：

```text
claim
ack
retry
dead-letter queue
```

Agent 不可能永远成功。

所以：

> **失败不是异常情况，而是系统设计的一部分。**

这也是 Agent 平台比简单 Chatbot 难很多的原因。

---

# 17. Agent 为什么尤其需要队列

普通 API：

```text
请求
↓
100ms
↓
响应
```

Agent：

```text
请求
↓
模型调用
↓
shell
↓
build
↓
test
↓
模型再调用
↓
git diff
↓
可能10分钟
```

甚至：

```text
1小时
```

所以 Agent 特别适合：

```text
asynchronous job
```

用户：

```text
POST /tasks
```

返回：

```json
{
  "task_id": "task-101",
  "status": "queued"
}
```

然后：

```text
GET /tasks/task-101
```

得到：

```json
{
  "status": "running"
}
```

最后：

```json
{
  "status": "succeeded"
}
```

这就是企业 Agent API 的雏形。

---

# 18. Codex 与这种架构怎样结合

当前 Codex 官方 SDK 就明确定位于把 Codex 集成到：

```text
CI/CD
内部工具
自己的应用
```

并允许程序化启动、继续和恢复 coding thread。citeturn661093search18

因此我们最终可以把：

```text
Runner
```

内部的：

```text
run_agent(task)
```

替换成：

```text
Codex SDK
```

结构：

```text
Queue
 ↓
Runner
 ↓
Codex SDK
 ↓
Codex Runtime
 ↓
Shell / Files / MCP
```

而且 OpenAI Agents SDK 当前的 Shell Tool 已明确支持两种环境：

```text
local runtime
hosted container
```

也允许开发者提供自己的 shell executor。citeturn661093search1turn661093search3

这说明我们现在学习的：

```text
Runner
Container
Shell Executor
```

并不是脱离 Agent 的 DevOps 知识，而正是现代 Agent SDK 的实际执行基础。

---

# 19. Kimi Code 如何对应

Kimi Code CLI 官方当前把自己定义为运行在 terminal 中、能读写代码、执行 shell、搜索网页并自主规划调整动作的 Agent。citeturn661093search2

它当前的插件工具也明确采用：

```text
独立 subprocess
→ stdout 返回给 Agent
```

这样的执行形式。citeturn661093search8

而截至 **2026 年 8 月 19 日**，Kimi Code CLI 官方 changelog 最新记录到 `0.37.2`，其中仍在修 context compaction 重试和 datasource tool 完整响应等运行时问题。citeturn661093search11

这提醒我们：

```text
LLM
```

只是系统的一部分。

Agent 实际可靠性还受：

```text
tool execution
context management
retry
state persistence
```

影响。

---

# 20. Codex / Kimi / 自研平台今天的对应关系

|层|Codex / OpenAI|Kimi Code|我们的 Mini Platform|
|---|---|---|---|
|Agent loop|Codex/Agents SDK|Kimi Runtime|后面自研|
|Shell|ShellTool / Codex runtime|Shell tool|runner.py|
|执行环境|local / hosted container|终端/subprocess|Docker runner|
|任务入口|SDK/API/CLI|CLI/SDK|API service|
|长任务调度|由应用设计|由应用设计|Redis queue|
|持久状态|由集成系统管理|session/runtime状态|PostgreSQL|
|扩展|MCP/Skills/tools|MCP/Skills/plugins|后面接MCP/Skills|

这里要注意：

**我没有说 Codex/Kimi 内部一定使用 Redis/PostgreSQL 这种架构。**

今天讲的是：

> 当你把 Coding Agent 能力做成企业服务时，一个合理的外围平台架构可以怎么设计。

要严格区分：

```text
公开产品实现事实
```

和：

```text
我们的平台架构设计
```

这也是以后读技术文档必须养成的习惯。

---

# 21. 今日 15 分钟实验

如果已经有 Docker：

先创建最小：

```yaml
services:

  redis:
    image: redis:8-alpine

  postgres:
    image: postgres:18-alpine

    environment:
      POSTGRES_PASSWORD: agent
      POSTGRES_USER: agent
      POSTGRES_DB: agentdb

    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:

  postgres_data:
```

启动：

```bash
docker compose up -d
```

查看：

```bash
docker compose ps
```

日志：

```bash
docker compose logs postgres
```

进入 Redis：

```bash
docker compose exec redis redis-cli
```

执行：

```text
PING
```

应该：

```text
PONG
```

退出。

进入 PostgreSQL：

```bash
docker compose exec postgres \
  psql -U agent -d agentdb
```

执行：

```sql
SELECT version();
```

然后：

```sql
CREATE TABLE tasks (
    id text primary key,
    status text not null
);
```

插入：

```sql
INSERT INTO tasks
VALUES ('task-101', 'queued');
```

查询：

```sql
SELECT * FROM tasks;
```

如果做到这里，你已经亲手建立了：

```text
Agent Task Persistence
```

的最小数据库。

---

# 22. 今日企业故障案例

### 现象

API：

```text
could not connect to server
Connection refused
localhost:5432
```

但：

```bash
docker compose ps
```

PostgreSQL 明明正常。

不要先怀疑数据库。

排查：

```text
API运行在哪里？
 ↓
Container

连接：
localhost:5432
 ↓
localhost是谁？
 ↓
API container自己
```

正确通常应该：

```text
postgres:5432
```

于是问题属于：

```text
Docker networking / service discovery
```

而不是：

```text
PostgreSQL故障
```

---

另一个现象：

```text
API偶尔启动失败：

connection refused postgres:5432

重启一下就好了
```

可能：

```text
Compose启动postgres container
↓
立即启动API
↓
Postgres仍初始化
↓
API连接失败
```

属于：

```text
startup race
```

修：

```text
healthcheck
+
service_healthy
+
应用自身retry
```

而不是简单：

```text
sleep 20
```

因为：

```text
sleep 20
```

只是猜测。

---

# 23. 今天第一次建立“服务化故障树”

以后 Agent Platform 出问题：

```text
Task一直queued
```

不要问模型。

故障树：

```text
Task queued
 │
 ├─ API成功写任务了吗？
 │
 ├─ Redis有消息吗？
 │
 ├─ Runner活着吗？
 │
 ├─ Runner连接Redis了吗？
 │
 ├─ Consumer group正常吗？
 │
 ├─ Runner拿到task了吗？
 │
 ├─ PostgreSQL状态更新成功吗？
 │
 └─ Agent真正开始执行了吗？
```

而：

```text
Task running但永远不结束
```

才继续：

```text
LLM
Shell
process tree
timeout
container
network
Git
```

这样故障定位就开始具有层次。

---

## 岗位能力映射

今天实际训练的是：

**Docker Compose、multi-service architecture、service discovery、container DNS、port、environment、volume、healthcheck、queue、worker、Redis Streams、PostgreSQL transaction、stateless runner。**

这已经对应：

```text
Agent Platform Engineer
AI Backend Engineer
Agent Deployment Engineer
DevOps / SRE
```

的基础能力。

其中最重要的三个工程判断是：

> **`localhost` 永远要先问“站在谁的网络空间里？”**

> **Container started 不代表 Service ready。**

> **Agent 长任务应该被视为 Job，而不是普通 HTTP 请求。**

### 今日检查题

1. 为什么 API container 中访问 PostgreSQL 通常应该写 `postgres:5432` 而不是 `localhost:5432`？
2. 为什么 `depends_on` 不能简单等价为“数据库已经 ready”？
3. Redis Queue 与 PostgreSQL 在 Agent 平台里分别适合承担什么职责？
4. 为什么 Agent Runner 最好尽量设计成 stateless？
5. 一个 Runner 取得任务后崩溃，为什么队列不能简单认为任务已经完成？

参考答案核心：**Compose 服务通过内部 DNS 名通信；启动容器和服务可用是两个阶段；Redis偏任务流和临时协调，PostgreSQL偏持久业务状态；无状态 Runner 才容易替换、扩容、恢复；任务需要 ACK/重试机制，不能以“已经分配”为“已经完成”。**

下一课继续这个小项目，进入 **HTTP/REST + SSE/WebSocket + Agent 流式输出**：为什么 Agent 任务适合异步提交，但用户又希望实时看到“正在读文件、正在执行测试、正在生成结果”，以及 Codex/Kimi 一类 CLI 的流式事件怎样映射到企业 Web API。