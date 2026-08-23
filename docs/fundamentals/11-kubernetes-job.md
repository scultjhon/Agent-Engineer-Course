# 第 11 课：Kubernetes 第一课——为什么 Agent API 用 Deployment，而 Coding Agent 任务更适合 Job

前面已经把 Mini Agent Platform 做到了：

```text
Nginx → API → Redis → Runner → PostgreSQL
                         ↓
                    Agent / Shell
```

今天把 Docker Compose 的思维迁移到 entity["software","Kubernetes","container orchestration platform"]。重点不是背 YAML，而是回答企业 Agent 部署最重要的问题之一：

> **哪些组件应该长期活着，哪些 Agent workload 应该“任务来了创建、完成后销毁”？**

Kubernetes 官方对两者的定义非常适合这个问题：Deployment 管理一组持续运行的 Pod，并维持期望状态；Job 则代表“运行到完成然后停止”的一次性任务。citeturn0search2turn0search1

## 1. 先把 Compose 翻译成 Kubernetes

上一阶段是：

```text
Docker Compose

api container
redis container
postgres container
runner container
```

迁移后不要机械变成“四个永远运行的 Pod”。

更合理的第一版是：

```text
Kubernetes Cluster
│
├── Deployment: agent-api
│      ├── Pod api-1
│      └── Pod api-2
│
├── Service: agent-api
│
├── Deployment/Stateful workload: redis
│
├── PostgreSQL
│
└── Job: agent-task-101
       │
       └── Pod
            ├── Agent Runtime
            ├── Git workspace
            ├── shell
            └── tests
```

其中最大的变化是：

```text
以前：
一个Runner一直运行
→ 从Redis拿很多任务

现在还可以：
一个Agent Task
→ 一个Job
→ 一个隔离Pod
→ 做完退出
```

两种模式都存在，但 Coding Agent 尤其值得理解后一种。

---

## 2. Pod 到底是什么

很多初学者把：

```text
Pod = Container
```

这是不准确的。

更接近：

```text
Pod
│
├── Container A
├── Container B
│
├── shared network namespace
└── shared volumes
```

最简单的 Pod 通常只有一个 container：

```text
Pod
└── agent-runner container
```

但以后 Agent workload 可以出现：

```text
Agent Pod
│
├── agent container
│
├── telemetry sidecar
└── shared workspace volume
```

所以 Kubernetes 真正调度的基本工作负载单位是：

```text
Pod
```

而不是直接调度某个 Docker container。

Pod 又是短暂资源。Kubernetes 官方明确提醒，不应把单个 Pod 当作可靠、永久存在的实体；Pod 可以被创建和销毁。citeturn0search0

这对 Agent 是好事：

```text
Task 101
↓
Pod 101
↓
执行
↓
结束
```

我们本来就不希望它永久存在。

---

# 3. 为什么 API 用 Deployment

Agent API 是：

```text
POST /tasks
GET /tasks/{id}
GET /tasks/{id}/events
```

它应该：

```text
一直在线
```

一个 API Pod 如果崩溃，我们希望：

```text
Kubernetes
↓
发现实际Pod数量少于期望值
↓
创建新Pod
```

这就是 Deployment 擅长的事情。

例如：

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: agent-api

spec:
  replicas: 2

  selector:
    matchLabels:
      app: agent-api

  template:
    metadata:
      labels:
        app: agent-api

    spec:
      containers:
        - name: api
          image: mini-agent-api:0.1
          ports:
            - containerPort: 8000
```

核心不是 YAML，而是：

```text
replicas: 2
```

表达：

> 我希望任何时候都有两个 API Pod。

Kubernetes Deployment Controller 会持续把实际状态向这个声明的期望状态调整；它也支持 rollout、扩缩容和 rollback。citeturn0search2

这叫：

```text
Desired State
```

---

# 4. Kubernetes 和 `docker run` 最大的思维差异

Docker：

```bash
docker run agent-api
```

你的思维容易变成：

> “启动这个进程。”

Kubernetes：

```yaml
replicas: 2
```

思维变成：

> “系统应该始终有两个健康实例。”

例如：

```text
Desired:
2

Actual:
2
```

正常。

某个 Pod crash：

```text
Desired:
2

Actual:
1
```

Controller：

```text
发现差异
↓
创建Pod
↓

Desired:
2

Actual:
2
```

这就是：

```text
reconciliation loop
```

这个思想非常重要，因为后面你自己写 Agent Orchestrator，也会出现类似模式：

```text
Task desired state:
running

Actual:
worker disappeared

↓
reconcile

重新调度
```

所以 Kubernetes Controller 思维和 Agent Platform Controller 思维其实高度相关。

---

# 5. 那为什么 Coding Agent Task 不直接也用 Deployment

假设任务：

> 修复 `HttpClient.cs` 的 timeout bug，运行测试后提交 patch。

它的生命周期是：

```text
start
↓
clone / checkout
↓
LLM
↓
tool calls
↓
tests
↓
result
↓
exit
```

这是典型：

```text
run-to-completion
```

如果使用 Deployment：

```text
Agent完成
↓
process exit 0
↓
Deployment发现Pod没了
↓
再启动一个
↓
Agent任务又执行一次
```

这显然不是我们想要的。

Job 正好相反。Kubernetes 官方定义：

> Job 创建一个或多个 Pod，并持续追踪它们，直到指定数量的 Pod 成功结束；最简单的 Job 就是可靠地运行一个 Pod 直到完成。citeturn0search1

因此：

```text
Long-lived service
→ Deployment

Run-to-completion task
→ Job
```

这是今天必须形成的判断。

---

# 6. 第一个 Agent Job

```yaml
apiVersion: batch/v1
kind: Job

metadata:
  name: agent-task-101

spec:
  template:
    spec:

      restartPolicy: Never

      containers:

        - name: agent

          image: mini-agent-runner:0.1

          env:
            - name: TASK_ID
              value: "task-101"
```

执行：

```bash
kubectl apply -f agent-job.yaml
```

观察：

```bash
kubectl get jobs
```

再：

```bash
kubectl get pods
```

可能看到：

```text
agent-task-101-x7abc   Running
```

一会儿：

```text
agent-task-101-x7abc   Completed
```

Job：

```text
0/1
↓
1/1
```

完成。

Kubernetes Job 对 Pod 失败也提供重试机制；如果 Pod 因节点故障等原因消失，Job Controller 可以重新创建 Pod。citeturn0search1

---

# 7. 这里出现 Agent 一个很危险的问题：Retry

假设 Agent：

```text
修改代码
↓
git push
↓
Pod突然挂掉
```

Job Controller：

```text
发现任务未成功完成
↓
重新创建Pod
```

新的 Agent：

```text
再次运行
↓
再次git push？
```

这就是分布式系统非常重要的：

```text
retry
+
side effect
```

问题。

因此不能简单认为：

```text
Kubernetes Job自动retry
= Agent可靠性解决了
```

反而要问：

```text
这个Agent步骤能重复执行吗？
```

例如：

```text
git status
```

重复通常问题不大。

```text
read file
```

问题也不大。

但：

```text
git push
创建PR
发送邮件
删除云资源
数据库UPDATE
发布生产版本
```

都具有：

```text
side effect
```

。

所以以后会正式学习：

```text
idempotency
```

即：

> 同一个操作重复执行，系统仍保持可接受的一致结果。

这正是 Agent 工程比“调用一次模型”复杂很多的地方。

---

# 8. `restartPolicy` 与 Job 重试不是一回事

Job 允许：

```text
restartPolicy:
Never
```

或者：

```text
OnFailure
``` citeturn0search1


要区分两层：

```text
Container失败
↓
同一个Pod内部是否restart

和

Pod失败
↓
Job Controller是否再创建Pod
```

所以以后看到：

```text
Agent运行了两次
```

不能只查：

```text
restartPolicy
```

还要查：

```text
Job backoff
Pod restart count
controller events
queue retry
应用自身retry
```

企业系统经常存在：

```text
LLM SDK retry
+
Agent runtime retry
+
Redis retry
+
Kubernetes Job retry
```

四层重试。

如果设计不好：

```text
1次故障
↓
3 × 3 × 3 × 3
↓
81次尝试
```

这叫：

```text
retry amplification
```

对 LLM 特别危险，因为还意味着：

```text
Token Cost Amplification
```

---

# 9. Service 又解决什么

Deployment 创建的 Pod：

```text
agent-api-f8d7c
agent-api-q91jk
```

可能随时变化。

每个 Pod 也有自己的 IP。

客户端不能写：

```text
http://10.244.2.17:8000
```

因为这个 Pod 明天可能不存在。

所以：

```text
Service
```

提供稳定网络抽象。

例如：

```yaml
apiVersion: v1
kind: Service

metadata:
  name: agent-api

spec:

  selector:
    app: agent-api

  ports:
    - port: 80
      targetPort: 8000
```

于是内部其它服务只访问：

```text
http://agent-api
```

Service：

```text
agent-api
       │
       ├── Pod A
       └── Pod B
```

Pod 换了：

```text
Pod A deleted
↓
Pod C created
```

客户端仍然：

```text
http://agent-api
```

不用修改。

这正是 Kubernetes Service 的核心目的：Pod 是 ephemeral 的，而 Service 提供稳定的网络入口与服务发现抽象。citeturn0search0turn0search4

---

# 10. Compose Service 和 Kubernetes Service 不完全是一回事

前面 Docker Compose：

```yaml
services:
  api:
```

这里：

```text
service
```

主要是 Compose 应用模型里的组件。

Kubernetes：

```yaml
kind: Service
```

是一个明确的 Kubernetes API object，主要解决：

```text
稳定endpoint
+
service discovery
+
traffic routing
```

不要因为名字一样就混为一谈。

---

# 11. Agent 为什么尤其需要资源限制

普通 Agent 一旦开始：

```text
npm install
dotnet build
pytest
ripgrep
LLM context construction
```

资源使用可能突然升高。

更危险：

```bash
while true; do ...; done
```

或者测试程序内存泄漏。

如果没有限制：

```text
Agent Pod
↓
吃光Node RAM
↓
影响API
↓
影响Redis
↓
影响其他Agent
```

所以 Pod container 可以定义：

```yaml
resources:

  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "2"
    memory: "2Gi"
```

Kubernetes 官方区分：

```text
requests
```

与：

```text
limits
```

并支持 CPU、memory、ephemeral storage 等资源的 requests/limits。citeturn0search6

先这样理解：

```text
request
≈ 调度时“我至少需要多少”

limit
≈ “最多允许我用多少”
```

虽然实际 CPU 和 memory 的 enforcement 行为还有细节，后面专门讲。

---

# 12. 为什么 Agent 比普通 Web API 更需要 limit

API：

```text
通常：
接请求
查数据库
返回
```

Agent：

```text
模型决定下一步
↓
模型可能运行build
↓
build又启动几十个process
↓
测试又可能并行
```

即：

```text
resource consumption
```

具有更大的：

```text
dynamic / model-directed
```

特征。

因此 Agent Platform 不能只控制：

```text
LLM Token Limit
```

还必须控制：

```text
CPU
RAM
Disk
Process count
Execution time
Network
```

最终一个任务预算会变成：

```text
Agent Budget

Token:
100k

Cost:
$2

Wall Time:
20 min

CPU:
2 cores

Memory:
4 GiB

Disk:
10 GiB

Network:
allowlist

Tool calls:
500
```

这才是真正的：

```text
Agent Resource Governance
```

。

---

# 13. 这与 Codex Sandbox 有什么关系

entity["software","OpenAI Codex","coding agent"] 的公开 system card 对 cloud agent 的描述非常适合拿来对照：云端任务运行在隔离 container 中，默认关闭网络访问，并限制其与用户宿主环境及 workspace 外敏感数据的交互；本地环境则依据操作系统采用不同 sandbox 机制。citeturn0search35turn0search37

因此我们可以把自己的 Kubernetes Agent workload 理解为：

```text
Kubernetes Pod
        ↓
Container
        ↓
Agent Sandbox
        ↓
Agent Runtime
        ↓
Shell
```

注意：

```text
Kubernetes
≠
Codex sandbox
```

它们属于不同层。

例如：

```text
K8s:
Pod最多2GiB RAM

Sandbox:
只能写/workspace

Agent Policy:
git push需要approval
```

三者同时工作。

这就是前面一直强调的：

```text
Defense in Depth
```

。

---

# 14. 一个企业 Agent Pod 最终可能长这样

```text
Agent Job Pod
│
├── Security
│   ├── non-root UID
│   ├── restricted filesystem
│   └── reduced capabilities
│
├── Resources
│   ├── CPU limit
│   ├── memory limit
│   └── ephemeral storage limit
│
├── Workspace
│   └── /workspace
│
├── Network Policy
│   └── allowed endpoints only
│
├── Secrets
│   └── short-lived credentials
│
└── Agent Runtime
    ├── model
    ├── shell
    ├── git
    ├── MCP
    └── tests
```

以后我们会逐项实现，而不是今天一次塞完。

---

# 15. API 如何动态创建 Job

最终 API 不应该让管理员手工：

```bash
kubectl apply -f task101.yaml
```

而是：

```text
POST /tasks
↓
API创建task
↓
Scheduler
↓
调用Kubernetes API
↓
Create Job
```

伪代码：

```python
def start_task(task):

    job = build_job_spec(
        task_id=task.id,
        image="agent-runner:0.1",
    )

    kubernetes.create_job(job)
```

于是：

```text
用户：
“修复Bug”

↓ HTTP

Task API

↓ Kubernetes API

Job task-101

↓ scheduler

Node 2

↓ Pod

Agent开始执行
```

这里第一次真正出现：

```text
Agent Orchestration
```

。

---

# 16. 不要让 Agent 自己随便访问 Kubernetes API

一个很危险的设计：

```text
Agent Pod
↓
拥有cluster-admin
↓
kubectl
```

那么 Agent 可以：

```text
删除Pod
读取Secret
创建高权限Pod
修改Deployment
```

这和前面：

```text
Agent + docker.sock
```

本质非常类似。

都是：

> 给 Agent 一个能够控制宿主编排系统的高权限接口。

更合理的是：

```text
Agent Pod
    │
    │ 没有K8s管理权限
    ▼

执行自己的任务
```

而：

```text
Scheduler / Controller
```

单独拥有：

```text
create Job
get Job
delete Job
```

等必要权限。

这就是后面要正式学习的：

```text
ServiceAccount
RBAC
Role
RoleBinding
```

。

---

# 17. 这里第一次理解 Control Plane

Kubernetes 本身有：

```text
Control Plane
```

负责：

```text
API
Scheduling
Controllers
Cluster state
```

Worker Node：

```text
真正运行Pod
```

而我们的 Agent Platform 也可以类比：

```text
Agent Control Plane
│
├── Task API
├── Scheduler
├── Policy
├── Approval
├── LLM routing
└── Audit

Agent Data/Execution Plane
│
└── Agent Jobs
    ├── shell
    ├── git
    ├── build
    └── tests
```

这个分离非常重要。

因为：

```text
模型产生的、不完全可信的执行
```

应该尽可能放在：

```text
Execution Plane
```

而不是让它直接控制：

```text
Control Plane
```

。

---

# 18. 今日真实 Agent 对比：Codex 与 OpenCode

entity["software","OpenCode","open-source coding agent"] 与 Codex 都可以最终执行 shell、编辑文件，但“Agent permission”与“基础设施隔离”应该继续分开理解。

例如：

```text
Agent层：

shell command
→ ask / allow / deny
```

下面仍然可以：

```text
Kubernetes Job
↓
non-root container
↓
resource limit
↓
network restriction
```

这意味着即使 Agent policy 出现配置错误：

```text
误允许了一个危险shell
```

基础设施仍可以限制：

```text
它最多影响这个task pod
```

。

Codex 云端公开设计同样体现了“Agent 在隔离环境中执行、网络默认受限”的思路。citeturn0search35

因此企业 Agent 平台绝不能把：

```text
Agent自己的permission UI
```

当成全部安全边界。

---

# 19. 今日 10～15 分钟实验

如果本机已有 Docker Desktop，可以启用它自带的 Kubernetes；也可以后面使用 `kind` 或 `minikube`。

今天只做一个最小实验。

建立：

```text
agent-job.yaml
```

内容：

```yaml
apiVersion: batch/v1
kind: Job

metadata:
  name: agent-lab

spec:

  template:

    spec:

      restartPolicy: Never

      containers:

        - name: agent

          image: alpine:latest

          command:
            - /bin/sh
            - -c

          args:
            - |
              echo "Agent task started"
              id
              pwd
              sleep 5
              echo "Agent task completed"
```

执行：

```bash
kubectl apply -f agent-job.yaml
```

然后：

```bash
kubectl get jobs
```

以及：

```bash
kubectl get pods
```

找到 Pod 后：

```bash
kubectl logs <pod-name>
```

应该看到：

```text
Agent task started
...
Agent task completed
```

最后：

```bash
kubectl get jobs
```

观察：

```text
Complete
```

。

这一实验的重点不是 Alpine，而是亲眼看到：

```text
Job
↓
创建Pod
↓
运行Container
↓
进程退出0
↓
Pod Completed
↓
Job Complete
```

。

---

# 20. 再故意制造失败

把：

```sh
echo "Agent task completed"
```

后面加入：

```sh
exit 1
```

再执行一个新 Job。

观察：

```bash
kubectl get pods
```

你可能看到不止一个失败/重试相关 Pod，具体数量受 Job 配置影响。

再看：

```bash
kubectl describe job <job-name>
```

重点观察：

```text
Events
Pods Statuses
Failed
```

。

你第一次会真正看到：

```text
Controller在处理失败
```

而不只是：

```text
shell返回exit code 1
```

。

---

# 21. PowerShell 用户需要注意

官方 Kubernetes 教程特别提醒，很多示例采用 POSIX shell：

```bash
export POD_NAME=...
$(...)
```

这些不能原样复制到 PowerShell。citeturn0search3

例如 Bash：

```bash
POD=$(kubectl get pods -o name | head -1)
```

PowerShell 更接近：

```powershell
$pod = kubectl get pods `
  -o jsonpath='{.items[0].metadata.name}'
```

然后：

```powershell
kubectl logs $pod
```

这也是为什么课程会持续并行讲：

```text
Bash
PowerShell
```

——Agent 经常生成“语义正确但 shell 方言错误”的命令。

---

# 22. 今日企业故障案例：Agent 一直重新执行

现象：

```text
task-101本来应该执行一次

但日志发现：
执行了4次
```

不要先问模型：

> 为什么重复工作？

先建立故障树：

```text
模型内部retry？
        │
        ├─ No
        ↓
Agent Runtime retry？
        │
        ├─ No
        ↓
Queue redelivery？
        │
        ├─ No
        ↓
Job创建了几个Pod？
        │
        ↓
kubectl describe job
        │
        ↓
Pod为什么失败？
        │
        ├─ OOMKilled
        ├─ exit code != 0
        ├─ Node failure
        ├─ timeout
        └─ eviction
```

假设最终发现：

```text
Pod:
OOMKilled
```

那么真正链路可能是：

```text
Agent运行dotnet test
↓
Memory > limit
↓
Container被kill
↓
Pod失败
↓
Job重试
↓
Agent重新开始
```

真正原因是：

```text
resource limit
```

而不是：

```text
LLM hallucination
```

。

这正是整个课程最终要训练出的能力：

> **Agent“行为异常”不等于模型异常。**

---

# 23. 再看一个非常现实的问题：为什么 Job 完成后 Pod 还在

很多初学者会问：

```text
Job已经Complete
为什么kubectl get pods还能看到它？
```

这是正常的。

Kubernetes 官方说明，Job 完成后通常不会立刻删除完成的 Pod，这样仍然可以查看日志和诊断信息；旧 Job 可以由用户或自动清理机制之后删除。citeturn0search1

这对 Agent 非常有价值：

```text
Agent失败
↓
Pod已经停止
↓
仍可：
kubectl logs
kubectl describe
查看exit code
```

这就是：

```text
post-mortem debugging
```

。

以后生产系统会再把这些日志发送到：

```text
Loki / Elasticsearch / Cloud Logging
```

因为 Pod 最终还是会被删除。

---

# 24. 今天的平台架构升级

现在 Mini Agent Platform 已经可以设计成：

```text
                     Internet
                        │
                       TLS
                        │
                    Ingress
                        │
                    Service
                        │
                 ┌──────┴──────┐
                 │             │
              API Pod       API Pod
                 │
                 ▼
             Task Queue
                 │
                 ▼
          Agent Scheduler
                 │
          Kubernetes API
                 │
       ┌─────────┼──────────┐
       ▼         ▼          ▼
    Job 101    Job 102    Job 103
       │         │          │
      Pod       Pod        Pod
       │
     Agent
       │
 ┌─────┼────────┐
 Git   LLM     Shell
```

这已经非常接近企业 Agent 平台最核心的执行模型。

---

## 岗位能力映射

今天真正训练的是：

**Pod、Deployment、Service、Job、Controller、desired state、reconciliation、resource requests/limits、run-to-completion workload、Agent retry、side effect、idempotency、control plane 与 execution plane。**

面试如果问：

> 为什么企业 Coding Agent 每个任务适合 Kubernetes Job，而不是 Deployment？

你应该能回答：

> API、Gateway 等长期服务需要持续维持期望副本数，因此适合 Deployment；一次 Coding Agent 执行属于 run-to-completion workload，完成后应该停止，因此更符合 Job。Job 还能跟踪完成状态并处理 Pod 失败重试。不过 Agent 往往有 Git push、PR、数据库修改等副作用，所以不能简单依赖自动重试，还必须考虑幂等性、重试层级和审计。

这已经是 Agent Deployment Engineer 应该具备的判断。

### 今日检查题

1. Pod 和 Container 为什么不能简单画等号？
2. 为什么 Agent API 更适合 Deployment，而一次 Coding Task 更适合 Job？
3. Service 解决了 Pod 的什么问题？
4. Agent Pod 被 `OOMKilled` 后为什么可能导致任务重复执行？
5. 为什么不能直接给 Agent Pod 一个 `cluster-admin` ServiceAccount？

答案核心：**Pod 可以包含一个或多个共享网络/存储上下文的容器；Deployment维持长期服务的期望状态而Job运行到完成；Service为易变Pod提供稳定网络入口；OOM导致Pod失败后Job可能重试；cluster-admin 会让模型驱动的工具执行获得控制整个集群的高权限能力。**

下一课进入 **Kubernetes Agent 安全核心：Namespace + ServiceAccount + RBAC + ConfigMap + Secret + SecurityContext**。我们会实际设计两个身份：`agent-scheduler` 可以创建受限 Job，而 `agent-runner` **不能创建 Pod、不能读取集群 Secret**，并亲手验证一次 `kubectl auth can-i` 的 Allow/Deny。这会把前面 Linux UID、Docker non-root、Agent Allow/Ask/Deny 和 Kubernetes RBAC 第一次完整串成一条权限链。