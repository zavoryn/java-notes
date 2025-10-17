---
tags: [八股, 计算机网络, HTTP, 浏览器缓存]
status: 待复习
created: 2026-05-06
---

# HTTP 协议详解

> [!tip] 模块速记
> **HTTP演进：1.0短连接→1.1长连接+管线化(但队头阻塞)→2.0多路复用+头部压缩→3.0(QUIC消除TCP层队头阻塞)。状态码：1xx信息、2xx成功、3xx重定向、4xx客户端锅、5xx服务端锅。GET幂等安全，POST不幂等不安全。缓存=强缓存(Cache-Control/Expires)+协商缓存(ETag/Last-Modified)。**

---

## 一、HTTP 协议演进

### 1.1 版本全景

| 版本 | 年份 | 核心特性 | 连接方式 | 比喻 |
|------|------|----------|----------|------|
| HTTP/1.0 | 1996 | 每个请求一个 TCP 连接 | 短连接 | 每次买东西都重新去一个商店 |
| HTTP/1.1 | 1997 | 持久连接、管线化 | 长连接（Keep-Alive） | 去一个商店买多样东西，但排队结账 |
| HTTP/2.0 | 2015 | 多路复用、头部压缩、服务器推送、二进制分帧 | 单连接多流 | 一个收银台开多个通道同时结账 |
| HTTP/3.0 | 2022 | 基于 QUIC（UDP）、0-RTT | QUIC 连接 | 走绿色通道，VIP 免排队 |

### 1.2 HTTP/1.1 核心特性

**持久连接（Keep-Alive）**

```
HTTP/1.0：
  请求1 → 关闭连接 → 请求2 → 关闭连接 → 请求3 → 关闭连接
  （三次握手 × 3，四次挥手 × 3）

HTTP/1.1 Keep-Alive：
  三次握手 → 请求1 → 请求2 → 请求3 → 四次挥手
  （三次握手 × 1，四次挥手 × 1）
```

**管线化（Pipelining）**

允许在收到上一个响应之前就发送下一个请求，但是**响应必须按顺序返回**。

**队头阻塞问题（Head-of-Line Blocking）**：如果第一个请求处理慢，后面所有的响应都被堵住了。这是 HTTP/1.1 管线化的致命缺陷，也是催生 HTTP/2.0 的重要原因。

**比喻**：就像超市只有一个收银台——队伍里第一个人买了一大车东西还要优惠券核销，后面所有人都得等着。这就是队头阻塞。

### 1.3 HTTP/2.0 核心特性

**二进制分帧**：HTTP/1.1 是文本协议，HTTP/2.0 把数据切成二进制帧来传输，更高效。

**多路复用（Multiplexing）**：

```
HTTP/1.1：一个连接上，请求串行处理
  [====请求A====][====请求B====][====请求C====]

HTTP/2.0：一个连接上，多个 Stream 并行传输
  [==A==][=B=][====C====][==A==][=B=][==C==]
    Stream1  Stream2  Stream3 (交错传输)
```

- 同一个 TCP 连接上可以同时跑多个 Stream
- **解决了 HTTP/1.1 的应用层队头阻塞**

**头部压缩（HPACK）**：维护一张索引表，常见头部只传索引号。

**服务器推送（Server Push）**：请求 index.html 时，服务端预判客户端还需要 style.css 和 app.js，主动推过去。

**比喻**：HTTP/2.0 就像一个高效的快递分拣中心——所有包裹到一个大仓库（一个TCP连接），用传送带分拣（多路复用），包裹标签用简码（头部压缩），仓库还能预判你要什么提前打包（Server Push）。

### 1.4 HTTP/3.0 —— 基于 QUIC

HTTP/2.0 解决了**应用层**的队头阻塞，但**传输层（TCP）**的队头阻塞还在。QUIC 基于 UDP，在应用层实现可靠传输，彻底消除了传输层队头阻塞。详见 [[04-传输层协议对比/TCP vs UDP]]。

---

## 二、HTTP 报文结构

### 2.1 请求报文

```
POST /api/users HTTP/1.1              ← 请求行：方法 + URL + 版本
Host: www.example.com                 ← 请求头（Header）
Content-Type: application/json
Authorization: Bearer eyJhbG...
Content-Length: 35
                                      ← 空行（分隔头部和Body）
{"name": "张三", "age": 25}           ← 请求体（Body）
```

**请求行三要素：**
| 要素 | 说明 | 示例 |
|------|------|------|
| **方法** | 对资源的操作语义 | GET / POST / PUT / DELETE |
| **URL** | 资源路径 | `/api/users/123` |
| **版本** | HTTP 协议版本 | `HTTP/1.1` / `HTTP/2.0` |

### 2.2 响应报文

```
HTTP/1.1 200 OK                       ← 状态行：版本 + 状态码 + 原因短语
Content-Type: application/json        ← 响应头（Header）
Content-Length: 28
Cache-Control: max-age=3600
                                      ← 空行
{"id": 123, "name": "张三"}           ← 响应体（Body）
```

**状态行三要素：**
| 要素 | 说明 | 示例 |
|------|------|------|
| **版本** | HTTP 协议版本 | `HTTP/1.1` |
| **状态码** | 三位数字，表示结果 | `200` / `301` / `404` / `500` |
| **原因短语** | 状态码的文字描述 | `OK` / `Not Found` |

### 2.3 常见 Header 速查

| Header | 方向 | 说明 | 串联知识点 |
|--------|------|------|-----------|
| **Host** | 请求 | 目标域名（HTTP/1.1 必填） | 一台服务器可能托管多个网站，Host 决定访问哪个 |
| **Content-Type** | 请求/响应 | Body 的数据格式 | `application/json` / `text/html` / `multipart/form-data` |
| **Authorization** | 请求 | 认证信息 | 串联 [[07-网络应用技术/Cookie与认证]] 的 JWT/Cookie |
| **Cache-Control** | 响应 | 缓存策略 | 串联 [[#四、浏览器缓存策略]] |
| **Connection** | 请求 | `keep-alive` 保持连接 | 串联 [[02-TCP连接管理/TCP连接管理]] 的 Keep-Alive |
| **Location** | 响应 | 重定向目标 URL | 配合 301/302 状态码使用 |

**🔗 知识串联：**
- **串联 05 HTTP演进**：HTTP/2.0 把这些文本头部压缩成了二进制帧（HPACK），减少传输量
- **串联 06 HTTPS**：这些 Header 在 HTTPS 中是加密的——中间人看不到你请求了什么 URL
- **串联 07 Cookie**：`Set-Cookie` 和 `Cookie` 也是 Header，用于维持会话

**🔗 知识串联（模块间）：**
- **串联 06 HTTPS**：HTTP 之上的 TLS 加密解决了明文传输的窃听/篡改/冒充问题，详见 [[06-HTTPS与安全认证/HTTPS与TLS详解]]
- **串联 07 DNS与CDN**：HTTP 请求的目标 IP 由 DNS 解析得到，CDN 通过 DNS 调度就近节点，详见 [[07-网络应用技术/DNS与CDN]]
- **串联 07 Cookie与认证**：HTTP 是无状态协议，Cookie/Session/JWT 通过 Header 维持用户身份，CORS 通过 Header 控制跨域权限，详见 [[07-网络应用技术/Cookie与认证]]

> **面试怎么考：** "HTTP 请求报文和响应报文的结构？常见的 Header 有哪些？"
>
> **面试话术：** "HTTP 报文分三部分：**行 + 头部 + Body**。请求报文的行是「方法 URL 版本」（如 `POST /api/users HTTP/1.1`），响应报文的行是「版本 状态码 原因短语」（如 `HTTP/1.1 200 OK`）。头部和 Body 之间用空行分隔。常见 Header 包括 Host（目标域名）、Content-Type（Body 格式）、Authorization（认证）、Cache-Control（缓存策略）等。HTTP/2.0 通过 HPACK 对这些头部做了压缩。"

---

## 三、HTTP 状态码

### 2.1 完整分类表

| 分类 | 含义 | 常见状态码 | 记忆口诀 |
|------|------|-----------|----------|
| **1xx** | 信息，请求已接收，继续处理 | 100 Continue、101 Switching Protocols | 信息提示 |
| **2xx** | 成功 | 200 OK、201 Created、204 No Content | 成功搞定 |
| **3xx** | 重定向 | 301 永久、302 临时、304 Not Modified | 换个地址 |
| **4xx** | 客户端错误 | 400/401/403/404/405/429 | 你的锅 |
| **5xx** | 服务端错误 | 500/502/503/504 | 我的锅 |

> [!note] 状态码口诀
> ```
> 1xx：信息提示还没完
> 2xx：请求成功已搞定
> 3xx：换个地址再请求
> 4xx：客户端你犯的错
> 5xx：服务端我背的锅
> ```

### 2.2 面试高频状态码详解

| 状态码 | 含义 | 面试要点 |
|--------|------|----------|
| **301 Moved Permanently** | 永久重定向 | 浏览器缓存，搜索引擎更新索引 |
| **302 Found** | 临时重定向 | 浏览器不缓存，搜索引擎保留原 URL |
| **304 Not Modified** | 未修改 | 配合缓存机制（ETag/If-Modified-Since） |
| **401 Unauthorized** | 未认证 | "你是谁？请出示证件" |
| **403 Forbidden** | 禁止访问 | "我知道你是谁，但你没权限进这个房间" |
| **429 Too Many Requests** | 请求频率过高 | 被限流了 |

### 2.3 常考辨析

**301 vs 302**：301 永久（搜索引擎转权重），302 临时（保留原地址）

**401 vs 403**：401 "请出示证件"（未认证），403 "你没权限进"（已认证但无权限）

**502 vs 504**：502 网关收到**无效响应**（上游挂了），504 网关**等超时了**（上游太慢）

---

## 三、GET vs POST

### 3.1 核心区别

| 维度 | GET | POST |
|------|-----|------|
| **语义** | 获取资源 | 创建/提交资源 |
| **参数位置** | URL 查询字符串 | 请求体（Body） |
| **安全性** | 参数暴露在 URL/浏览器历史/日志中 | 参数在 Body 中，相对安全 |
| **幂等性** | 幂等（多次请求结果相同） | 不幂等（多次提交创建多条） |
| **缓存** | 可缓存 | 一般不可缓存 |

### 3.2 POST 请求的 Content-Type

> [!warning] 面试追问："POST 请求体有哪些数据格式？"

| Content-Type | 说明 | 典型场景 |
|--------------|------|----------|
| `application/x-www-form-urlencoded` | **默认**，key=value&key=value 格式 | 普通表单提交 |
| `multipart/form-data` | 分段传输，每段有自己的 Content-Type | **文件上传**（必须用这个） |
| `application/json` | JSON 格式 | **前后端分离 API**（最常用） |
| `text/plain` | 纯文本 | 很少用 |
| `application/xml` | XML 格式 | SOAP / 老系统 |

```
// 1. 表单默认（application/x-www-form-urlencoded）
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=zhangsan&password=123456

// 2. 文件上传（multipart/form-data）
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

(二进制文件内容)
------WebKitFormBoundary--

// 3. JSON（前后端分离最常用）
POST /api/users HTTP/1.1
Content-Type: application/json

{"name": "张三", "age": 25}
```

> **面试话术：** "POST 请求体的格式由 Content-Type 决定，常见三种：`application/x-www-form-urlencoded` 是表单默认格式（键值对），`multipart/form-data` 用于文件上传（分段传输），`application/json` 是前后端分离 API 最常用的格式。"

### 3.3 HTTP 基于 TCP 还是 UDP？

> [!note] 面试追问："HTTP 是基于 TCP 还是 UDP？"

**HTTP/1.0、1.1、2.0 基于 TCP**——因为需要可靠传输（网页数据不能丢）。

**HTTP/3.0 基于 UDP**——但不是裸 UDP，而是 QUIC 在 UDP 上实现了可靠传输（确认+重传），取了 UDP 的灵活（无队头阻塞、连接迁移）又补了可靠性。

```
HTTP/1.0, 1.1  →  TCP  →  IP
HTTP/2.0       →  TCP  →  IP
HTTP/3.0       →  QUIC →  UDP → IP
                     ↑
              应用层实现可靠传输
```

> **面试话术：** "HTTP/1.0 到 2.0 都基于 TCP，因为 HTTP 需要可靠传输。HTTP/3.0 改用 QUIC + UDP——QUIC 在 UDP 之上实现了类似 TCP 的可靠传输，同时解决了 TCP 的队头阻塞和连接迁移问题。"

### 3.4 语义层面——这才是面试关键

> **面试时不要说"GET 用 URL 传参，POST 用 Body 传参"**——这太浅了。

**正确回答**：

"GET 和 POST 的本质区别在**语义层面**，而不是技术实现层面。

- **GET 是安全的（Safe）和幂等的（Idempotent）**：只读操作，不会改变服务端资源状态。所以可以被缓存、可以预加载。
- **POST 不是安全的也不是幂等的**：用于提交数据，会改变服务端状态。默认不被缓存。

至于参数放 URL 还是 Body，这是实现细节，不是本质区别。"

### 3.5 幂等性扩展

| 方法 | 是否幂等 | 说明 |
|------|----------|------|
| GET | 是 | 读多少次都一样 |
| PUT | 是 | 更新多少次结果一样 |
| DELETE | 是 | 删一次和删多次最终都是不存在 |
| POST | **否** | 多次提交创建多个资源 |
| PATCH | 否 | 可能每次结果不同 |

---

## 四、浏览器缓存策略

> [!warning] 这是高频面试题！几乎每场面试都会问到缓存策略。

### 4.1 缓存决策流程

```
浏览器请求资源
    │
    ▼
是否有本地缓存？─────── 无 → 向服务器请求 → 存入缓存 → 返回资源
    │
    有
    ▼
强缓存是否过期？─────── 未过期 → 直接使用本地缓存（200 from cache）
    │
    已过期
    ▼
协商缓存：向服务器验证资源是否更新
    │
    ▼
资源未修改？─────── 是 → 304 Not Modified → 使用本地缓存
    │
    否 → 服务器返回新资源（200）+ 新缓存头
```

### 4.2 强缓存

强缓存命中时**不发送请求到服务器**，直接从本地缓存取。

| 头部 | 说明 | 优先级 |
|------|------|--------|
| **Cache-Control** | `max-age=3600`（3600秒内有效）；`no-cache`（跳过强缓存，走协商）；`no-store`（不缓存）；`public`/`private` | **高**（HTTP/1.1） |
| **Expires** | 绝对过期时间，如 `Expires: Wed, 21 Oct 2025 07:28:00 GMT` | **低**（HTTP/1.0，被 Cache-Control 覆盖） |

> [!note] Cache-Control: no-cache ≠ 不缓存
> `no-cache` 是"跳过强缓存，每次都要跟服务器协商验证"——不是完全不缓存！`no-store` 才是完全不缓存。

### 4.3 协商缓存

强缓存过期后，浏览器带验证信息向服务器确认资源是否更新。

| 验证方式 | 请求头 | 响应头 | 说明 |
|----------|--------|--------|------|
| **ETag** | `If-None-Match: "abc123"` | `ETag: "abc123"` | 资源的哈希值/版本标识，精确 |
| **Last-Modified** | `If-Modified-Since: Wed, 21 Oct 2025 07:28:00 GMT` | `Last-Modified: Wed, ...` | 资源最后修改时间，精度秒级 |

**ETag vs Last-Modified**：
- ETag 更精确（文件内容不变但修改时间变了，ETag 不会误判）
- Last-Modified 有精度限制（秒级，一秒内多次修改无法区分）
- **优先级**：ETag > Last-Modified

**比喻**：强缓存像"牛奶保质期"——没过期直接喝，不看产地；协商缓存像"牛奶产地查询"——保质期过了但不确定是否变质，打电话问牧场确认。

---

## 五、RESTful API 设计原则

### 5.1 核心原则

| 原则 | 说明 | 示例 |
|------|------|------|
| **资源导向** | URL 代表资源（名词），不代表动作 | `/users` 而不是 `/getUser` |
| **HTTP 方法语义** | 用方法表达操作 | GET 读、POST 创建、PUT 更新、DELETE 删除 |
| **统一接口** | 标准化的 CRUD 操作 | 同一个 `/users` 资源支持多种方法 |
| **无状态** | 每个请求包含所有必要信息 | 不依赖服务端 Session |
| **HATEOAS** | 响应中包含相关资源链接 | 返回 JSON 中嵌入可操作的超链接 |

### 5.2 RESTful 方法映射

| HTTP 方法 | 操作 | 幂等 | 示例 URL |
|-----------|------|------|----------|
| GET | 获取资源列表/单个 | 是 | GET /users, GET /users/123 |
| POST | 创建新资源 | 否 | POST /users |
| PUT | 全量更新资源 | 是 | PUT /users/123 |
| PATCH | 部分更新资源 | 否 | PATCH /users/123 |
| DELETE | 删除资源 | 是 | DELETE /users/123 |

> [!warning] 面试加分
> RESTful 不是强制规范，而是设计风格。实际项目中不必完全遵循（如批量删除用 POST + body 更合理），但理解其核心思想——**资源 + 方法语义 + 无状态**——能体现你的设计素养。

---

## 六、WebSocket 基本概念

### 6.1 WebSocket 是什么？

WebSocket 是一种在**单个 TCP 连接**上进行**全双工通信**的协议，弥补了 HTTP 只能客户端主动请求的缺陷。

### 6.2 WebSocket 与 HTTP 的关系

| 对比 | HTTP | WebSocket |
|------|------|-----------|
| **通信方向** | 半双工（客户端请求→服务端响应） | 全双工（双方随时发） |
| **连接模式** | 请求-响应，短连接或长连接 | 持久连接，建立后一直保持 |
| **开销** | 每次请求带完整头部 | 建立时带头部，之后只发数据帧（2~10字节头部） |
| **适用场景** | 普通请求-响应 | 实时推送（聊天、股票行情、游戏） |

### 6.3 WebSocket 建立过程

WebSocket 通过 HTTP Upgrade 机制建立：

```
客户端发送：
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: xxx

服务端响应：
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: yyy

→ 之后切换为 WebSocket 协议，全双工通信
```

> [!tip] 面试加分
> WebSocket 建立连接用的是 HTTP（101 Switching Protocols），但建立后不再遵循 HTTP 请求-响应模式，而是独立的双向通信协议。

### 6.4 SSE vs WebSocket vs 长轮询

> [!warning] 面试热点（美团直考 + LLM 流式输出场景）
> "WebSocket 和 SSE 的区别？ChatGPT 的流式输出用什么？"

| 维度 | **长轮询（Long Polling）** | **SSE（Server-Sent Events）** | **WebSocket** |
|------|---------------------------|-------------------------------|---------------|
| **方向** | 客户端请求→服务端挂起→有数据才回 | **服务端→客户端**单向推送 | **双向**全双工 |
| **协议** | 普通 HTTP | 普通 HTTP | 独立协议（WS/WSS） |
| **实现复杂度** | 最简单 | 简单 | 较复杂 |
| **连接开销** | 每次都要重新发 HTTP 请求 | 一次 HTTP 连接持续推送 | 一次握手持续通信 |
| **数据格式** | 任意 | **纯文本**（text/event-stream） | 文本 + 二进制 |
| **断线重连** | 需自己实现 | **浏览器原生自动重连** | 需自己实现 |
| **浏览器兼容** | 全部 | 大部分（IE 不支持） | 大部分 |
| **典型场景** | 早期即时通讯 | **股票行情、新闻推送、LLM 流式输出** | 聊天室、游戏、协同编辑 |

```
长轮询（Long Polling）：
  客户端 → GET /messages → 服务端（没新消息？先挂着不回）
                                    ...
                                  有新消息了！
  客户端 ← 200 OK + 数据 ← 服务端
  客户端 → 立刻再发 GET /messages → （循环）

SSE（Server-Sent Events）：
  客户端 → GET /events (Accept: text/event-stream) → 服务端
  客户端 ← data: {"msg": "hello"}     ← 服务端推送
  客户端 ← data: {"msg": "world"}     ← 服务端推送
  客户端 ← data: [DONE]               ← 服务端推送结束
  （ChatGPT 就是用 SSE 做流式输出！）

WebSocket：
  客户端 ←→ 服务端（建立后双方随时发数据）
```

> **面试话术：** "三者都是实现服务端推送的技术。长轮询最简单但开销大——每次都要重新发 HTTP 请求。SSE 基于 HTTP，服务端单向推送，浏览器原生支持断线重连，ChatGPT 的流式输出就用 SSE——每生成一个 token 就推一个 data 事件给前端。WebSocket 是全双工双向通信，适合聊天室、游戏等需要双方频繁交互的场景。选型原则：单向推送用 SSE，双向交互用 WebSocket，兜底用长轮询。"

**🔗 知识串联：**
- **串联 04 QUIC**：WebSocket 的低延迟特性和 QUIC 的 0-RTT 有互补性，WebSocket over HTTP/3 值得关注
- **串联 06 HTTPS**：WebSocket Secure（wss://）就是在 WebSocket 之上加了 TLS 加密

---

## 6.5 gRPC vs HTTP/REST

> [!tip] 面试考点（Java 后端微服务方向常考）："gRPC 和 HTTP/REST 有什么区别？什么时候用 gRPC？"

| 维度 | HTTP/REST | gRPC |
|------|-----------|------|
| **定位** | 通用应用层通信协议 | 高性能 RPC 框架（基于 HTTP/2） |
| **数据格式** | JSON / XML（文本、可读） | Protobuf（二进制、紧凑） |
| **底层协议** | HTTP/1.1 或 HTTP/2 | 强制 HTTP/2 |
| **多路复用** | HTTP/1.1 不支持；HTTP/2 支持 | 原生支持（基于 HTTP/2 Stream） |
| **接口约束** | 依赖文档约定（Swagger/OpenAPI） | `.proto` 文件强类型定义 + 自动生成代码 |
| **流式通信** | 不原生支持（需 SSE/WebSocket） | 原生支持双向流 |
| **性能** | JSON 序列化较慢、体积大 | Protobuf 快 5~10 倍、体积小 3~10 倍 |
| **生态** | 浏览器原生、通用性最强 | 需 SDK，浏览器支持有限（gRPC-Web） |
| **调试** | curl/Postman 直接调 | 需要 gRPC 工具（grpcurl） |
| **典型场景** | 对外 API、前后端交互、开放平台 | 微服务内部调用、跨语言服务通信 |

**gRPC 四种通信模式：**

```
1. Unary（一元调用）    ──  请求 → 响应  （类似普通 HTTP）
2. Server Streaming    ──  请求 → 流式响应  （大数据分批返回）
3. Client Streaming    ──  流式请求 → 响应  （文件分片上传）
4. Bidirectional      ──  双向流  （聊天/实时协作）
```

**Java 技术栈中的定位：**

```
对外接口（前端/第三方）：
  ┌─────────────────────────────────┐
  │  HTTP/REST + JSON（通用、可读）   │
  └─────────────────────────────────┘

对内服务调用（微服务之间）：
  ┌─────────────────────────────────┐
  │  gRPC + Protobuf（高性能、强类型）│
  └─────────────────────────────────┘

技术栈对比：
  Dubbo   → 自定义 RPC 协议（默认 Dubbo 协议）
  gRPC    → 基于 HTTP/2 + Protobuf
  Feign   → 基于 HTTP/REST（声明式客户端）
```

> **面试话术：** "gRPC 和 HTTP/REST 不是替代关系，而是分工不同。gRPC 基于 HTTP/2，使用 Protobuf 做序列化，性能高、体积小、支持双向流，适合微服务内部高频调用。HTTP/REST 用 JSON，通用性强、调试方便、浏览器原生支持，适合对外 API 和前后端交互。一般 Java 项目中，对外接口用 REST，内部服务调用用 gRPC 或 Dubbo。"

**🔗 知识串联：**
- **串联 01 HTTP/2**：gRPC 强制基于 HTTP/2，天然享受多路复用和头部压缩
- **串联 04 QUIC**：未来 gRPC 可能基于 HTTP/3（QUIC），获得更好的移动网络体验
- **串联 08 面试突击**：[[08-综合面试题/计网面试突击]] 中提到了 gRPC 在微服务中的应用

---

## 七、面试回答模板

> **面试官：HTTP/1.1、2.0、3.0 的区别？队头阻塞怎么解决？**

"HTTP/1.1 有 Keep-Alive 长连接和管线化，但管线化存在应用层队头阻塞——第一个请求慢后面全堵。HTTP/2.0 用多路复用（Stream 交错传输）解决了应用层队头阻塞，但 TCP 层的队头阻塞还在——一个 TCP 包丢了所有 Stream 都得等。HTTP/3.0 用 QUIC 基于 UDP 实现可靠传输，每个 Stream 独立，彻底消除了传输层队头阻塞。"

> **面试官：浏览器缓存策略是怎样的？**

"浏览器缓存分两层：**强缓存**和**协商缓存**。强缓存命中时不发请求，直接用本地缓存——由 Cache-Control（max-age）和 Expires 控制，前者优先级高。强缓存过期后走协商缓存：带 If-None-Match（ETag）或 If-Modified-Since（Last-Modified）去服务器验证，没变化返回 304 继续用本地缓存，有变化返回 200 + 新资源。ETag 精度比 Last-Modified 高，优先级也更高。"

---

## 相关面试题

| # | 题目 | 要点 |
|---|------|------|
| 1 | HTTP版本演进区别 | 1.0短连接→1.1长连接/管线化→2.0多路复用→3.0 QUIC |
| 2 | HTTP队头阻塞 | 应用层(HOL/1.1)→TCP层(HOL/2.0)→彻底消除(QUIC/3.0) |
| 3 | HTTP报文结构 | 请求行/状态行 + Header + 空行 + Body |
| 4 | HTTP状态码 | 1xx~5xx分类，301/302/304/401/403/502/504辨析 |
| 4 | GET vs POST | 语义层面：GET安全幂等，POST不安全不幂等 |
| 5 | 浏览器缓存策略 | 强缓存(Cache-Control/max-age)+协商缓存(ETag/Last-Modified) |
| 6 | RESTful设计原则 | 资源导向+方法语义+无状态 |
| 7 | WebSocket | HTTP Upgrade建立，全双工持久连接，实时推送场景 |
| 8 | SSE vs WebSocket vs 长轮询 | 单向推送用SSE/双向用WS/兜底长轮询，ChatGPT用SSE |
| 9 | POST请求Content-Type | 表单默认urlencoded/文件用multipart/API用JSON |
| 10 | HTTP基于TCP还是UDP | 1.0~2.0基于TCP，3.0基于QUIC(UDP) |
| 11 | gRPC vs HTTP/REST | gRPC=HTTP/2+Protobuf高性能RPC，对外REST对内gRPC |

---

> **相关笔记**：[[06-HTTPS与安全认证/HTTPS与TLS详解]] | [[04-传输层协议对比/TCP vs UDP]] | [[07-网络应用技术/从URL到页面]] | [[07-网络应用技术/DNS与CDN]] | [[07-网络应用技术/Cookie与认证]] | [[计算机网络总览]]