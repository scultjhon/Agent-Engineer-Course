# 第 10 课：Nginx / Reverse Proxy / DNS / TLS——为什么本地 SSE 正常，上线后却“一次性吐完”

上一课我们已经有了：

```text
Browser
  ↓
FastAPI /events
  ↓
SSE
  ↓
Agent Runner
```

但生产环境里一般不会直接让用户访问：

```text
http://server-ip:8000
```

而会变成：

```text
https://agent.example.test
        ↓
       DNS
        ↓
      Nginx
        ↓
   FastAPI:8000
        ↓
   Agent Platform
```

今天真正要理解的是：**Nginx 并不只是“转发端口”，它会介入连接、Header、Buffer、Timeout 和 TLS。SSE 最容易就在这里被破坏。**

---

## 1. Reverse Proxy 到底是什么

假设 FastAPI 跑在：

```text
127.0.0.1:8000
```

用户访问：

```text
https://agent.example.test/tasks/101/events
```

真正过程：

```text
Browser
   │
   │ HTTPS :443
   ▼
 Nginx
   │
   │ HTTP :8000
   ▼
FastAPI
```

Nginx 官方 `ngx_http_proxy_module` 的作用就是把请求转发到另一个服务器；官方最小例子就是：

```nginx
location / {
    proxy_pass http://localhost:8000;
}
``` citeturn641670view0


因此：

```text
Browser认为自己只在和Nginx通信
```

而：

```text
FastAPI看到的是Nginx转过来的请求
```

这就是：

**Reverse Proxy（反向代理）**。

---

## 2. 为什么企业不直接暴露 FastAPI:8000

主要不是因为 FastAPI “不能联网”。

而是 Nginx 可以统一承担：

```text
TLS/HTTPS
域名入口
路由
Header处理
连接管理
限流
日志
负载均衡
静态文件
超时策略
```

于是未来可以：

```text
/api/
   ↓
Agent API

/events/
   ↓
SSE Gateway

/
   ↓
Web前端
```

全部通过同一个：

```text
https://agent.example.test
```

进入。

这就是所谓：

```text
Edge / Gateway Layer
```

的雏形。

---

# 3. `proxy_pass` 是最核心的指令

例如：

```nginx
server {
    listen 80;

    location /api/ {
        proxy_pass http://api:8000;
    }
}
```

如果 Nginx 也在上一课的 Docker Compose network 内：

```text
Compose Network

nginx
 │
 ├── api:8000
 ├── redis:6379
 └── postgres:5432
```

那么：

```text
api
```

可以直接作为 Docker 内部 DNS 名。

这里把前面知识串起来了：

```text
Docker service discovery
+
Nginx reverse proxy
```

---

# 4. Host Header 为什么重要

Nginx 官方示例还写了：

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
``` citeturn641670view0


为什么？

浏览器原本访问：

```text
agent.example.test
```

如果 Nginx 不把原始 Host 信息正确传给后端，后端可能只看到类似：

```text
api:8000
```

这会影响：

```text
生成回调URL
安全校验
日志
多租户域名
OAuth redirect
```

同样：

```text
X-Real-IP
X-Forwarded-For
X-Forwarded-Proto
```

常用于告诉后端：

```text
真实客户端是谁
原请求是不是HTTPS
```

所以生产应用经常会遇到：

> 后端明明通过 HTTPS 访问，为什么它生成的链接却是 `http://`？

很可能不是应用逻辑本身，而是：

```text
Reverse proxy headers
```

没有处理好。

---

# 5. SSE 最大的坑：Buffering

上一课我们的 FastAPI 每秒：

```text
yield event
```

本地：

```bash
curl -N http://localhost:8000/events
```

可以看到：

```text
event 1

一秒后
event 2

一秒后
event 3
```

但经过 Nginx 后可能：

```text
等待几十秒
↓
1、2、3、4、5 一次性全部出来
```

为什么？

Nginx 官方文档非常明确：

```text
proxy_buffering
默认 = on
```

开启时 Nginx 会尽可能快地从 upstream 收取响应，然后放进 buffer；如果关闭 buffering，则收到 upstream 数据后会同步地立即传给 client。citeturn641670view0

这就是上一课那个现象：

```text
LLM正在stream
↓
FastAPI也在yield
↓
Browser却没有实时显示
```

很可能就是：

```text
Nginx Buffer
```

---

# 6. SSE Location 常见配置

可以专门给 SSE endpoint：

```nginx
location /events/ {
    proxy_pass http://api:8000;

    proxy_http_version 1.1;

    proxy_buffering off;
    proxy_cache off;

    proxy_read_timeout 3600s;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

今天最需要看懂：

```nginx
proxy_buffering off;
```

不是“提升性能”的万能设置。

而是对这种：

```text
small incremental events
```

来说，我们明确要求：

> 收到多少就尽快转发多少。

Nginx 也支持 upstream 用：

```text
X-Accel-Buffering: no
```

响应头动态关闭 buffering。官方文档明确说明了这一点。citeturn641670view0

MDN 的 SSE 示例同样直接发送：

```text
X-Accel-Buffering: no
```

并使用 `text/event-stream`。citeturn352097search1

---

# 7. 所以 FastAPI 也可以主动告诉 Nginx

例如：

```python
return StreamingResponse(
    event_stream(),
    media_type="text/event-stream",
    headers={
        "Cache-Control": "no-cache",
        "X-Accel-Buffering": "no",
    },
)
```

这样结构变成：

```text
FastAPI:
X-Accel-Buffering: no
        ↓
Nginx看到
        ↓
不要缓存这个response
```

但企业配置里，我更希望你能理解：

```text
应用层
+
代理层
```

两边都必须检查。

不能只改一边然后猜。

---

# 8. `proxy_read_timeout` 又是什么

假设 Agent 一段时间没有任何输出：

```text
Running integration test...
```

持续 5 分钟。

连接：

```text
Browser ↔ Nginx
```

还在。

但：

```text
Nginx ↔ FastAPI
```

长时间没收到新数据。

Nginx 有：

```text
proxy_read_timeout
```

控制读取 upstream 响应的超时。citeturn641670view0

所以会出现：

```text
Agent后台仍在执行
↓
SSE连接突然断了
↓
浏览器自动重连
```

这并不意味着：

```text
Agent crash
```

而可能只是：

```text
proxy timeout
```

又一次体现：

> **看到连接断，不要直接定位到模型。**

---

# 9. SSE Heartbeat

MDN 提到，SSE 可以发送以 `:` 开头的 comment line，用来防止连接因空闲而超时。citeturn352097search1

例如服务器每 15 秒：

```text
: ping

```

客户端不会把它当普通业务 event。

但是中间代理会看到：

```text
连接仍然有数据
```

所以 Agent Platform 可以设计：

```text
每15秒无业务事件
↓
发送heartbeat
```

例如：

```python
yield ": heartbeat\n\n"
```

这样可以降低：

```text
proxy
load balancer
firewall
```

因为连接长期 idle 而关闭的概率。

---

# 10. Event ID 与断线重连

SSE 还有：

```text
id:
```

字段。

例如：

```text
id: 101
event: tool.started
data: {...}

id: 102
event: tool.output
data: {...}
```

MDN 说明 `id` 会更新 EventSource 的 last event ID，而连接丢失后 EventSource 默认会尝试重新建立连接。citeturn352097search1turn352097search0

这对企业 Agent 很有价值：

```text
Browser断线
↓
重新连接
↓
告诉Server:
我最后收到event 102
↓
Server从103继续
```

否则：

```text
断线重连
↓
用户漏掉了一段Agent执行过程
```

后面讲 Event Store / Redis Streams 时，我们可以真正把这个能力做出来。

---

# 11. DNS 到底负责什么

现在用户不是访问：

```text
123.45.67.89
```

而是：

```text
agent.example.test
```

DNS 主要负责：

```text
域名
↓
找到目标
```

最常见两个记录先记住：

```text
A
CNAME
```

Cloudflare 当前 DNS 文档中：

```text
A
```

用于把名称指向 IPv4 地址；

```text
CNAME
```

则用于把一个名称指向另一个 hostname。citeturn641670view3

例如：

```text
agent.example.test
    A
    ↓
203.0.113.10
```

于是：

```text
Browser
↓
DNS查询
↓
203.0.113.10
↓
连接443
```

---

# 12. DNS 不等于 HTTP 转发

这点很重要。

DNS 只解决：

```text
agent.example.test在哪
```

它不负责：

```text
/api走FastAPI
/events走SSE
/static走前端
```

这些是：

```text
Nginx / Reverse Proxy
```

决定的。

所以：

```text
DNS
↓
定位主机

Nginx
↓
定位服务
```

先这样记即可。

以后 Kubernetes 会进一步变成：

```text
External DNS
↓
Load Balancer
↓
Ingress/Gateway
↓
Service
↓
Pod
```

今天就是这个链路的最小版。

---

# 13. TLS / HTTPS 到底增加了什么

HTTP：

```text
Browser
↓
明文HTTP
↓
Server
```

HTTPS：

```text
Browser
↓
TLS
↓
Encrypted HTTP
↓
Server
```

TLS 主要解决三类问题：

```text
加密
身份验证
完整性
```

也就是：

```text
别人不容易直接看内容
↓
浏览器确认自己连接的是目标站点
↓
通信内容不应被无声篡改
```

对于 Agent 平台特别重要，因为流量里可能有：

```text
source code
prompt
model output
API data
tool results
user identifiers
```

所以生产环境把 Agent API 裸奔在 HTTP 上通常不可接受。

---

# 14. Certificate 是干什么的

服务器需要：

```text
TLS certificate
```

简单理解：

```text
证书
   │
   ├── 这个公钥属于哪个域名
   └── 哪个CA对此进行了签名/验证
```

Let's Encrypt 官方说明，其 ACME 流程会验证你是否控制某个域名，然后签发证书；证书之后可以由 Web Server 用于 HTTPS。citeturn641670view2

所以大致：

```text
你：
我控制 agent.example.test

↓ ACME challenge

CA验证

↓

签发证书

↓

Nginx加载certificate/private key

↓

https://agent.example.test
```

后面部署到 Kubernetes 时，这一套会升级成：

```text
Ingress/Gateway
+
cert-manager
+
Let's Encrypt
```

---

# 15. TLS Termination

生产结构很常见：

```text
Browser
 │
 │ HTTPS
 ▼
Nginx
 │
 │ HTTP
 ▼
FastAPI
```

在 Nginx：

```text
HTTPS解密
```

这叫：

```text
TLS termination
```

FastAPI 和 Nginx 在同一台主机/受控内网时，可以由 Nginx 统一管理公网证书。

也可以：

```text
Browser
↓ HTTPS
Nginx
↓ HTTPS
Backend
```

这叫：

```text
TLS re-encryption
```

是否需要取决于部署安全边界。

不要形成：

> “HTTPS 就只是在代码里改成 https://”。

其实背后存在：

```text
证书
private key
TLS handshake
termination
trust
renewal
```

一整套基础设施。

---

# 16. Nginx + Agent 的最小生产拓扑

把上一课项目升级：

```text
Internet
   │
   │ https://agent.example.test
   ▼
 DNS
   │
   ▼
 Nginx
   │
   ├── /api/*
   │       ↓
   │     API
   │
   └── /events/*
           ↓
        SSE API
           ↓
       Event Bus
           ↓
       Agent Runner
```

Docker Compose 可以：

```yaml
services:

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"

  api:
    build: ./api

  runner:
    build: ./runner

  redis:
    image: redis:8-alpine

  postgres:
    image: postgres:18-alpine
```

这样真正暴露 host 的主要只有：

```text
80
443
```

而：

```text
8000
5432
6379
```

可以只留在内部 Docker Network。

这就是：

```text
减少攻击面
```

。

---

# 17. 今天第一次看 Nginx 配置

最简单开发版：

```nginx
events {}

http {

    server {
        listen 80;

        location /api/ {
            proxy_pass http://api:8000/;
        }

        location /events/ {
            proxy_pass http://api:8000/events/;

            proxy_http_version 1.1;
            proxy_buffering off;
            proxy_cache off;

            proxy_read_timeout 3600s;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

今天不要背。

只把它翻译成人话：

```text
收到HTTP请求
↓
如果路径是 /api/
↓
交给 api:8000

如果路径是 /events/
↓
也交给api
↓
但是关闭response buffering
↓
允许长时间读取
```

这就够了。

---

# 18. 一个经典坑：`proxy_pass` 尾部 `/`

Nginx：

```nginx
location /api/ {
    proxy_pass http://api:8000/;
}
```

与：

```nginx
location /api/ {
    proxy_pass http://api:8000;
}
```

URI 拼接行为可能不同。

这是 Nginx 初学阶段非常容易让 Agent API 出现：

```text
404
```

的原因。

例如浏览器：

```text
/api/tasks
```

后端实际得到：

```text
/tasks
```

还是：

```text
/api/tasks
```

需要根据 `location` + `proxy_pass` 配置判断。

以后遇到：

```text
直接访问FastAPI正常
Nginx后404
```

第一反应不要看模型。

检查：

```text
location匹配
proxy_pass目标
upstream实际path
```

。

---

# 19. 一个 Agent Streaming 真实故障树

现象：

```text
localhost:8000/events
每秒输出

通过：
https://agent.example.test/events
30秒后一次性出现
```

排查：

```text
Agent Runtime在实时发event吗？
          ↓
FastAPI在实时yield吗？
          ↓
curl直接访问FastAPI是否实时？
          ↓
Nginx proxy_buffering？
          ↓
X-Accel-Buffering？
          ↓
上层Load Balancer有没有buffer？
          ↓
浏览器是否实时消费？
```

如果：

```text
直接FastAPI正常
经过Nginx失败
```

排查范围已经大幅缩小。

这是非常重要的：

```text
分层验证
```

能力。

---

# 20. 第二个故障：SSE 几分钟就断

现象：

```text
Agent仍运行
↓
Web显示Disconnected
```

排查：

```text
FastAPI crash？
↓
Nginx proxy_read_timeout？
↓
Load Balancer idle timeout？
↓
Firewall/NAT timeout？
↓
heartbeat有没有？
↓
浏览器EventSource是否重连？
```

MDN 明确说明 EventSource 连接关闭时默认会重新连接；服务器也可以通过 `retry` 控制重连等待时间。citeturn352097search1

所以：

```text
重连
```

不应该视为异常设计之外的事情。

而应该是 SSE 系统正常需要处理的生命周期。

---

# 21. 第三个故障：HTTPS 后 OAuth/回调 URL 变成 HTTP

架构：

```text
Browser
↓ HTTPS
Nginx
↓ HTTP
FastAPI
```

FastAPI 看到：

```text
自己的socket是HTTP
```

于是如果没有正确处理：

```text
X-Forwarded-Proto: https
```

它可能认为：

```text
外部请求也是HTTP
```

最后生成：

```text
http://agent.example.test/callback
```

结果：

```text
OAuth redirect mismatch
```

所以：

```text
Reverse Proxy Headers
```

不是“锦上添花”。

它会影响应用对于：

```text
scheme
client IP
host
```

的认知。

---

# 22. 安全：为什么不能信任任意 `X-Forwarded-For`

如果客户端自己发送：

```text
X-Forwarded-For: 127.0.0.1
```

而你的后端：

```text
无条件相信
```

它可能错误认为：

```text
请求来自本机
```

所以生产系统必须明确：

```text
哪些Proxy是trusted proxy
```

然后只信任这些代理添加/重写的 forwarded headers。

这是后面：

```text
Zero Trust
Ingress
API Gateway
```

都会继续出现的基础问题。

---

# 23. Nginx 也会成为 Agent 平台的观测点

以后 access log：

```text
request
status
latency
upstream latency
bytes sent
client IP
```

可以帮助判断：

```text
Agent事件端点是否大量502？
SSE平均连接多久？
哪个upstream最慢？
有没有异常流量？
```

所以以后可观测性架构：

```text
Browser
↓
Nginx access log
↓
API trace
↓
Runner trace
↓
LLM call
↓
Tool execution
```

会串成：

```text
end-to-end trace
```

。

这就是为什么我们后面要学 OpenTelemetry。

---

# 24. 今日 10～15 分钟实验

如果上一课有 FastAPI SSE：

### 第一步：直接访问

```bash
curl -N http://localhost:8000/tasks/task-101/events
```

观察是不是：

```text
一条一条出来
```

### 第二步：加 Nginx

建立：

```text
nginx.conf
```

```nginx
events {}

http {
    server {
        listen 8080;

        location /events/ {
            proxy_pass http://host.docker.internal:8000/;

            proxy_buffering off;
            proxy_cache off;
            proxy_read_timeout 3600s;
        }
    }
}
```

如果 macOS/Windows Docker Desktop：

```bash
docker run --rm \
  -p 8080:8080 \
  -v "$PWD/nginx.conf:/etc/nginx/nginx.conf:ro" \
  nginx:alpine
```

然后：

```bash
curl -N http://localhost:8080/events/tasks/task-101/events
```

目标不是把 URL 配得多漂亮。

而是亲自感受：

```text
Browser/curl
↓
Nginx
↓
FastAPI
↓
Streaming
```

这条链。

---

# 25. 再做一个很有价值的对照实验

先：

```nginx
proxy_buffering off;
```

观察。

然后临时改：

```nginx
proxy_buffering on;
```

重新：

```bash
nginx -s reload
```

或者重启 container。

再观察 streaming 行为。

是否一定明显变化取决于事件大小、刷新行为和具体环境，但这个实验能让你真正意识到：

> **Proxy 可以改变应用数据到达用户的时间特性。**

这对 Agent streaming 很重要。

---

# 26. 企业 Agent 平台现在长什么样了

到今天为止，我们已经构出了：

```text
                      Internet
                         │
                         ▼
                        DNS
                         │
                         ▼
                    HTTPS / TLS
                         │
                         ▼
                       Nginx
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
           REST                     SSE
             │                       │
             ▼                       │
         API Service                 │
             │                       │
             ▼                       │
        Redis Queue                  │
             │                       │
             ▼                       │
       Agent Runner ─── Events ──────┘
             │
      ┌──────┼────────┐
      ▼      ▼        ▼
     LLM    Git      Shell
                      │
                      ▼
                   Docker
             │
             ▼
         PostgreSQL
```

前 10 课看起来学了很多不同东西：

```text
Git
Shell
Process
Linux permission
Docker
Compose
HTTP
SSE
Nginx
DNS
TLS
```

实际上全部是在回答同一个问题：

> **一个 Coding Agent 从模型的一次决策，到最终安全、稳定地在企业环境执行并让用户看到结果，中间到底经过了哪些工程层？**

---

## 岗位能力映射

今天对应的是真实：

**Agent Deployment Engineer / Backend Engineer / Platform Engineer / DevOps / SRE**

技能：

```text
Reverse Proxy
proxy_pass
Forwarded Headers
SSE buffering
timeout
heartbeat
DNS
A/CNAME
TLS/HTTPS
certificate
TLS termination
service exposure
```

面试如果问：

> 本地 Agent Streaming 正常，但上线经过 Nginx 后变成最后一次性返回，你怎么排查？

你现在至少应该回答：

> 先确认 Provider/Runner 是否实时产生事件，再绕过 Nginx 直接访问 upstream SSE endpoint。如果 upstream 正常而代理后异常，重点检查 Nginx `proxy_buffering`、`X-Accel-Buffering`、代理或负载均衡的缓存/缓冲，以及客户端消费方式；如果是连接中途断开，再检查 `proxy_read_timeout`、上层 idle timeout 和 heartbeat，而不是直接认为模型没有 streaming。

这已经是一个比较像工程师的答案。

### 今日检查题

1. DNS 和 Nginx 分别解决什么问题？
2. 为什么 Nginx 默认 buffering 对普通 API 可能没问题，但对 SSE 可能产生明显影响？
3. 为什么 Agent 正在运行，却可能出现 SSE 连接断开？
4. `X-Forwarded-Proto` 为什么会影响 HTTPS 后端行为？
5. 为什么“用了 Docker + Nginx + HTTPS”仍不能说明 Agent 平台已经安全？

答案核心：**DNS 找主机，Nginx 找后端服务；SSE要求增量及时传递而 proxy buffering 可能聚合响应；proxy/load-balancer idle/read timeout 都可能断连接；反向代理 TLS termination 后需要把原始 scheme 告诉后端；安全还取决于 Agent 权限、容器、Secret、MCP、网络、身份认证、RBAC、输入和工具边界等多层控制。**

下一课进入 **Kubernetes 第一课：Pod / Deployment / Service / Job**。我们会把现在的 Docker Compose 架构迁移成最小 K8s 架构，并重点讨论为什么 **API 应该是 Deployment，而一次 Coding Agent 执行任务更适合 Job/短生命周期 Pod**，从而真正进入企业 Agent 部署工程主线。