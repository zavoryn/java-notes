---
tags:
  - 八股
  - AI
  - Agent
status: 待复习
created: 2026-05-16
---

# MCP 协议（Model Context Protocol）

> MCP 是 AI Agent 工具调用的标准化协议，相当于"AI 的 Type-C 接口"。本文从核心定义、架构组件、交互流程、技术规范到安全优化全面讲解。

---

## 一、知识点讲解

### 1. MCP 核心定义

MCP（Model Context Protocol）是一个**开放、通用的应用层协议**，专为 AI 模型与外部工具、数据源的交互而设计。

核心类比：**MCP 是"AI 的 Type-C 接口"**

| 对比维度 | Type-C 接口 | MCP 协议 |
|---------|------------|---------|
| 解决的问题 | 充电/数据传输接口碎片化（Micro-USB、Lightning...） | AI 与工具交互格式碎片化 |
| 核心价值 | 一个接口统一所有设备的连接 | 一个协议统一所有工具的接入 |
| 标准化程度 | USB-IF 组织维护标准 | Anthropic 主导，社区开放 |
| 生态效应 | 设备厂商只需实现一次 Type-C | 工具开发者只需实现一次 MCP Server |

### 2. MCP 解决的五大核心问题

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP 之前的世界（碎片化）                        │
│                                                                 │
│  LLM ──格式A──> 工具X                                           │
│  LLM ──格式B──> 工具Y        每个工具需要单独适配                   │
│  LLM ──格式C──> 工具Z        集成代码量大、复用性差                 │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                    MCP 之后的世界（统一化）                        │
│                                                                 │
│  LLM ──MCP标准──> MCP Client ──MCP标准──> MCP Server(工具X/Y/Z) │
│                                                                 │
│  一次实现，处处可用；上下文统一管理；错误码标准化                    │
└─────────────────────────────────────────────────────────────────┘
```

| # | 核心问题 | 具体表现 | MCP 的解法 |
|---|---------|---------|-----------|
| 1 | **交互格式碎片化** | 各家工具交互格式各异，每个对接需单独适配 | 统一 JSON-RPC 2.0 消息格式 |
| 2 | **上下文管理混乱** | 多轮交互缺乏统一上下文管理，历史信息传递不规范 | Host 统一管理上下文生命周期 |
| 3 | **版本兼容复杂** | 工具侧与大模型侧版本更新频繁，缺乏版本协商 | 内置协议版本协商机制 |
| 4 | **错误处理不统一** | 错误码和错误信息格式各异，问题排查困难 | 标准 JSON-RPC 2.0 错误码 |
| 5 | **资源利用率低** | 缺乏高效连接管理和资源调度，GPU 利用率约 40% | 连接复用、按需调用、批量处理 |

### 3. MCP 三大核心组件详解

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                   MCP Host（宿主应用）                  │   │
│  │   - 管理多个 Client 连接                                │   │
│  │   - 统一上下文管理（创建/更新/清理）                      │   │
│  │   - 应用层业务逻辑                                      │   │
│  │   典型实现：Claude Desktop / VS Code AI 插件             │   │
│  │                                                        │   │
│  │   ┌──────────────┐   ┌──────────────┐                 │   │
│  │   │  MCP Client  │   │  MCP Client  │   ...           │   │
│  │   │  (客户端A)    │   │  (客户端B)    │                 │   │
│  │   └──────┬───────┘   └──────┬───────┘                 │   │
│  └──────────┼──────────────────┼──────────────────────────┘   │
│             │                  │                              │
│    ┌────────▼────────┐  ┌─────▼─────────┐                   │
│    │  MCP Server A   │  │  MCP Server B  │                   │
│    │  (数据库工具)    │  │  (文件系统工具)  │                   │
│    │                 │  │                │                   │
│    │  - 资源查询(GET) │  │  - 工具操作    │                   │
│    │  - 工具操作(POST)│  │  - Prompt注入  │                   │
│    │  - Prompt注入    │  │                │                   │
│    └─────────────────┘  └────────────────┘                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### MCP Host（宿主应用）—— "中枢调度中心"

| 职责 | 说明 |
|------|------|
| 连接管理 | 管理多个 MCP Client 连接，协调不同工具的交互时序 |
| 上下文管理 | 统一管理上下文的创建、封装、更新、过期清理 |
| 业务逻辑 | 接收用户请求，分发至对应 Client/Server，汇总结果返回 |
| 生命周期 | 管理整个会话的上下文清理与生命周期 |

典型实现：Claude Desktop、VS Code AI 插件、企业级 AI 助手后端服务

#### MCP Client（客户端）—— "翻译官"

| 职责 | 说明 |
|------|------|
| 连接建立 | 与 MCP Server 建立加密连接（HTTPS/WebSocket） |
| 请求封装 | 将应用请求、上下文按 MCP 标准格式封装发送 |
| 响应解析 | 接收 Server 响应，按标准格式解析后传递给 Host |
| 状态维护 | 维护会话状态与重试逻辑，处理网络异常和超时 |

#### MCP Server（工具服务端）—— "能力暴露者"

| 职责 | 说明 |
|------|------|
| 能力暴露 | 暴露 3 类核心能力：资源查询(GET)、工具操作(POST)、Prompt 注入 |
| 请求处理 | 解析 Client 的标准化请求，验证合法性后调用本地工具 |
| 响应封装 | 将工具执行结果按 MCP 标准格式封装返回 |
| 版本协商 | 支持协议版本协商和错误标准化反馈 |

### 4. MCP 核心交互流程（6 步）

```
用户请求 ──→ ①上下文创建 ──→ ②请求标准化 ──→ ③解析执行 ──→ ④响应封装
                                                              │
                                                              ▼
用户响应 ←── ⑥生命周期管理 ←── ⑤响应解析与上下文更新 ←────────┘
```

| 步骤 | 执行者 | 具体操作 |
|------|--------|---------|
| ① 上下文创建与封装 | Host | 收集用户需求、历史对话、场景配置，按 MCP 标准格式封装，生成会话 ID、角色标识 |
| ② 请求标准化与发送 | Client | 封装为 MCP 标准消息（消息 ID、发送者、接收者、消息类型、内容），通过 Stdio/HTTP+SSE/WebSocket 发送 |
| ③ 请求解析与工具执行 | Server | 解析消息类型与内容，验证请求合法性（权限、参数格式），调用本地工具执行 |
| ④ 响应封装与回传 | Server | 将结果按 MCP 标准格式封装，添加标准化错误码（若失败），返回给 Client |
| ⑤ 响应解析与上下文更新 | Client → Host | Client 解析响应数据传递给 Host；Host 更新上下文（添加返回信息、延长过期时间） |
| ⑥ 上下文清理与生命周期管理 | Host | 会话结束时清理过期上下文，释放资源 |

### 5. MCP 技术规范

#### 5.1 消息结构（JSON 字段）

```json
{
  "message_id": "msg_abc123",
  "sender": "client_001",
  "receiver": "server_db",
  "timestamp": 1715836800000,
  "performative": "request",
  "content": {
    "tool": "query_table",
    "args": { "table": "users", "columns": ["id", "name"], "limit": 100 }
  },
  "metadata": {
    "session_id": "sess_xyz",
    "protocol_version": "1.0"
  }
}
```

| 字段 | 说明 |
|------|------|
| `message_id` | 消息唯一标识，用于请求-响应关联 |
| `sender` | 发送者标识（Client/Server 的 ID） |
| `receiver` | 接收者标识 |
| `timestamp` | 时间戳 |
| `performative` | 消息类型：`request` / `response` / `notification` |
| `content` | 消息内容（工具名、参数、结果等） |
| `metadata` | 元数据（会话 ID、协议版本等） |

#### 5.2 错误处理（JSON-RPC 2.0 错误码）

| 错误码 | 英文 | 含义 | 类比 HTTP |
|--------|------|------|----------|
| -32700 | Parse error | 消息解析失败 | 400 Bad Request |
| -32600 | Invalid Request | 无效请求格式 | 400 Bad Request |
| -32601 | Method not found | 调用的方法不存在 | 404 Not Found |
| -32602 | Invalid params | 参数格式错误 | 422 Unprocessable |
| -32603 | Server error | 服务器内部错误 | 500 Internal Error |

#### 5.3 传输方式对比

| 应用场景 | 传输方式 | 核心优势 | 平均延迟 | 适用场景 |
|----------|----------|----------|----------|---------|
| 本地集成 | **Stdio** | 零网络开销，部署简单 | ~1ms | 本地命令行工具、单机部署 |
| 远程调用 | **HTTP + SSE** | 穿透防火墙，兼容性强 | 20-100ms | 跨服务器工具对接、企业内网 |
| 实时交互 | **WebSocket** | 双向低延迟，持续推送 | 5-20ms | 多模态实时响应、流式输出 |

```
本地工具 ──── Stdio（标准输入输出）────→ MCP Server（同进程）
远程工具 ──── HTTP+SSE（请求-流式响应）──→ MCP Server（远程）
实时工具 ──── WebSocket（双向长连接）───→ MCP Server（远程）
```

### 6. MCP 应用实例：智能数据分析助手

#### 完整调用链

```
用户输入 "分析最近30天用户增长趋势"
    │
    ▼
┌─────────┐    ┌─────────────┐    ┌──────────────┐    ┌─────────┐
│   Host   │───→│   Client    │───→│ MCP Server   │───→│ 数据库   │
│ (Claude  │    │ (请求封装)   │    │ (db-mcp-     │    │ (MySQL) │
│ Desktop) │    │             │    │  server)     │    │         │
└─────────┘    └─────────────┘    └──────────────┘    └─────────┘
    │                                                      │
    │  1. 识别意图 → 选择工具                                │
    │  2. 调用 list_tables 查看表结构                        │
    │  3. 调用 query_table 查询数据                          │
    │  4. 分析结果 → 生成报告                                 │
    │                                                      │
    ▼
用户收到分析报告
```

#### Step 1：MCP Server 定义工具（TypeScript）

```typescript
// tools.ts —— 防止模型直接写SQL，定义结构化安全接口
export const tools = [
  {
    name: "list_tables",
    description: "列出当前数据库的表",
    inputSchema: {
      type: "object",
      properties: {}
    }
  },
  {
    name: "query_table",
    description: "对指定表执行只读 SQL 查询",
    inputSchema: {
      type: "object",
      properties: {
        table: { type: "string" },
        columns: {
          type: "array",
          items: { type: "string" }
        },
        limit: { type: "number", default: 100 }
      },
      required: ["table", "columns"]
    }
  }
];
```

#### Step 2：实现 MCP Server

```typescript
import { Server } from "@modelcontextprotocol/sdk/server";
import { tools } from "./tools";
import { db } from "./db";

const server = new Server({ name: "db-mcp-server" });

server.setTools(tools);

server.onToolCall(async (tool, args) => {
  if (tool === "list_tables") {
    return await db.listTables();
  }
  if (tool === "query_table") {
    return await db.select(args.table, args.columns, args.limit);
  }
});

server.start();
```

#### Step 3：LLM 自动编排（无需手工编码）

用户输入 "分析一下最近 30 天用户增长趋势"，LLM 内部自动完成：
1. 识别用户意图 → 选择合适的 MCP 工具
2. 调用 `list_tables` 查看表结构
3. 调用 `query_table` 查询数据
4. 分析结果并生成报告返回用户

### 7. MCP 安全规范

| 安全维度 | 具体措施 | 类比后端 |
|---------|---------|---------|
| **传输安全** | HTTPS / TLS 1.3+ | 与 HTTPS/TLS 一致 |
| **认证授权** | OAuth 2.1 with PKCE | 类似 Spring Security OAuth2 授权码模式 |
| **凭证管理** | 安全存储与自动轮换 | 类似 Vault / KMS 密钥管理 |
| **审计日志** | 完整操作记录与追溯 | 类似操作审计日志系统 |

```
┌──────────┐    OAuth 2.1 PKCE    ┌──────────┐
│  Client  │ ←─────────────────→ │   Auth   │
│          │    获取Access Token   │  Server  │
└────┬─────┘                      └──────────┘
     │
     │  携带 Token 请求
     ▼
┌──────────┐                      ┌──────────┐
│  MCP     │    验证 Token         │  审计日志  │
│  Server  │ ──────────────────→  │  系统     │
└──────────┘                      └──────────┘
```

### 8. MCP 性能优化

| 优化策略 | 说明 | 效果 |
|---------|------|------|
| **上下文分片（Context Sharding）** | 将大上下文拆分为多个分片，按需加载 | 减少单次传输数据量 |
| **连接复用** | 复用已建立的连接，避免重复握手 | 降低连接建立开销 |
| **按需调用** | 只在需要时调用工具，避免冗余调用 | GPU 利用率从 40% → 75% |
| **批量处理** | 将多个请求合并为一个批量请求 | 减少网络往返次数 |

---

## 二、串联理解记忆

### MCP vs Function Calling

这是面试高频考点，核心区别：**"模型怎么喊" vs "工具世界怎么被组织起来"**

| 维度 | Function Calling | MCP |
|------|-----------------|-----|
| **本质** | 模型输出结构化 JSON 的能力 | 工具调用的标准化协议 |
| **关注点** | 模型如何"喊"出工具调用 | 工具世界怎么被组织起来 |
| **作用层** | 模型推理层（模型内部） | 应用协议层（模型外部） |
| **标准** | 各厂商自定义（OpenAI/Google格式不同） | 统一开放标准 |
| **工具发现** | 预先硬编码在 Prompt 中 | 运行时动态发现 |
| **上下文** | 无统一上下文管理 | 统一上下文生命周期 |
| **生态** | 封闭在各自平台 | 开放生态，一次实现到处用 |

> **一句话记忆**：Function Calling 解决"模型怎么喊出工具调用"，MCP 解决"工具世界怎么被标准化地组织起来"。

### MCP vs Skill

| 维度 | MCP | Skill |
|------|-----|-------|
| 角色 | 工具的"驱动程序" | 能力的"操作手册" |
| 回答的问题 | **怎么做**（How） | **做什么、什么时候做**（What & When） |
| 类比 | JDBC 驱动 | 业务 Service 层 SOP |

协同关系：Skill 的 Steps 中调用工具时，底层走 MCP 协议。

### MCP 与 Agent 基础

在 [[八股/agent/1-Agent基础概念]] 中定义的 Agent 四大组件（感知/规划/记忆/执行），**MCP 是"执行"模块的标准化协议**：

```
Agent 核心架构:
  感知模块 → 规划模块 → 记忆模块 → 执行模块
                                    └──→ MCP 协议标准化
                                          ├── 工具发现
                                          ├── 上下文管理
                                          └── 响应格式统一
```

### MCP 与 Java 后端

| MCP 概念 | Java 后端对应 | 串联理解 |
|----------|-------------|---------|
| MCP 协议 | API 网关（如 Spring Cloud Gateway） | 统一入口，标准化路由 |
| OAuth 2.1 with PKCE | Spring Security OAuth2 授权码模式 | 几乎完全一致的认证流程 |
| JSON-RPC 2.0 | RESTful JSON API / gRPC | 消息格式标准化的思想一致 |
| Host/Client/Server | 网关/微服务客户端/微服务 | 分层解耦思想一致 |
| Stdio/HTTP/WebSocket | 进程内调用/HTTP/RPC | 传输方式选型策略一致 |

### 记忆口诀

> **MCP口诀**：
> **Host 管 Client 连，Client 封装发 Server，Server 暴露三能力（资源/工具/Prompt）。**
> **Function Calling 是"嘴"怎么喊，MCP 是"手"怎么接。**
> **三传 Stdio 本地快，HTTP 远程通，WebSocket 实时推。**

---

## 三、面试题

### Q1：MCP 协议是什么？

MCP（Model Context Protocol）是一个开放、通用的应用层协议，专为 AI 模型与外部工具、数据源的交互而设计。它可以类比为"AI 的 Type-C 接口"，通过统一的消息格式、上下文管理和错误处理规范，解决了工具交互碎片化的问题。核心价值包括：集成代码减少 80%、扩展能力提升 5 倍、GPU 利用率从 40% 提升至 75%。

### Q2：Function Calling 和 MCP 有什么区别？

Function Calling 是模型层面的一种能力，指模型能够输出结构化的 JSON 来"调用"一个函数——它解决的是"模型怎么喊出工具调用"。MCP 是应用层的标准化协议，解决的是"工具世界怎么被组织起来"——包括工具发现、上下文管理、消息格式、错误处理等。Function Calling 是"嘴"，MCP 是"手"。

### Q3：MCP 协议体现在什么地方？（四个层面）

| 层面 | 体现 |
|------|------|
| **格式层面** | 统一的 JSON-RPC 2.0 消息结构（message_id/sender/receiver/performative/content/metadata） |
| **交互层面** | 标准化的 Host → Client → Server 三层交互流程和 6 步生命周期 |
| **错误层面** | 统一的 JSON-RPC 2.0 错误码（-32700 ~ -32603） |
| **安全层面** | OAuth 2.1 with PKCE 认证、TLS 1.3+ 传输加密、审计日志 |

### Q4：MCP 的三大组件（Host/Client/Server）各自的职责？

- **Host（宿主应用）**：中枢调度中心，管理多个 Client 连接、统一上下文管理（创建/更新/清理）、应用层业务逻辑
- **Client（客户端）**：翻译官，与 Server 建立加密连接、封装标准请求、解析标准响应、维护会话状态
- **Server（工具服务端）**：能力暴露者，暴露三类能力（资源查询/工具操作/Prompt 注入）、解析执行请求、封装标准响应

### Q5：MCP 支持哪些传输方式？各自的适用场景？

| 传输方式 | 延迟 | 适用场景 |
|---------|------|---------|
| **Stdio** | ~1ms | 本地集成（命令行工具、同进程调用） |
| **HTTP + SSE** | 20-100ms | 远程调用（跨服务器、企业内网、穿透防火墙） |
| **WebSocket** | 5-20ms | 实时交互（多模态响应、流式输出、双向通信） |

### Q6：LangChain4j 有用到 MCP 吗？

LangChain4j 是 Java 生态的 LLM 开发框架，它已经集成了 MCP 协议支持。通过 `langchain4j-mcp` 模块，Java 开发者可以：
- 将 Java 方法/工具注册为 MCP Server
- 通过 MCP Client 连接外部 MCP 工具服务
- 在 Agent 的 Tool 调用链路中无缝使用 MCP 协议

这与 Java 后端的微服务架构非常契合——每个工具服务就是一个 MCP Server，Agent 框架充当 Host/Client。

### Q7：项目中哪里集成了 MCP 协议？

> 此题需要结合自己的项目回答，以下是通用思路：

1. **工具注册层**：将数据库查询、API 调用、文件操作等封装为 MCP Server
2. **Agent 框架层**：使用 LangChain4j 的 MCP Client 连接各工具服务
3. **上下文管理层**：通过 MCP Host 统一管理多轮对话的上下文传递
4. **安全层**：OAuth 2.1 认证 + 审计日志

### Q8：如何从零开发一个 MCP Server？

开发 MCP Server 的核心步骤：

**Step 1：确定能力**
- 明确 Server 提供哪些 Tools（工具）、Resources（资源）、Prompts（提示模板）
- 每个 Tool 需定义：名称、描述、输入参数的 JSON Schema

**Step 2：选择传输方式**
- 本地工具 → Stdio（进程间通信，最简单）
- 远程服务 → HTTP + SSE（支持跨网络访问）

**Step 3：实现核心方法**
```typescript
// 简化的 MCP Server 结构
const server = new Server({ name: "my-server", version: "1.0" });

// 注册工具
server.setRequestHandler(ListToolsRequest, async () => ({
  tools: [{
    name: "query_database",
    description: "查询数据库",
    inputSchema: { type: "object", properties: { sql: { type: "string" } } }
  }]
}));

// 处理工具调用
server.setRequestHandler(CallToolRequest, async (request) => {
  if (request.params.name === "query_database") {
    const result = await executeSQL(request.params.arguments.sql);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  }
});
```

**Step 4：注册到 MCP Client（Host 端配置）**
```json
{
  "mcpServers": {
    "my-database": {
      "command": "node",
      "args": ["./my-mcp-server.js"]
    }
  }
}
```

关键注意点：
- **Schema 设计要精确**：描述清晰、类型明确，LLM 靠这些信息决定何时调用
- **错误处理要友好**：返回结构化错误信息，而非抛异常
- **幂等性**：工具调用尽量幂等，LLM 可能重复调用

### Q9：MCP 的三种传输方式怎么选？

| 传输方式 | 原理 | 适用场景 | 优缺点 |
|---------|------|---------|-------|
| **Stdio** | 标准输入输出，进程间通信 | 本地工具、CLI 集成 | 简单可靠，但不能跨网络 |
| **HTTP + SSE** | HTTP 请求 + 服务端推送 | 远程服务、Web 应用 | 支持远程访问，SSE 适合流式 |
| **WebSocket** | 全双工持久连接 | 高频交互、Multi-Agent | 性能最好，但实现复杂 |

选型决策：
- **桌面应用集成本地工具**（如 Claude Desktop + 文件系统）→ Stdio
- **Web 应用集成远程 API**（如网页端 + 外部服务）→ HTTP + SSE
- **实时 Multi-Agent 通信**（Agent 间高频交互）→ WebSocket

实际项目中 Stdio 最常用（大多数 MCP Server 是本地进程），HTTP+SSE 用于需要远程访问的场景。

### Q10：多团队 MCP 架构如何设计？

企业级 MCP 部署需要考虑多个团队共享和隔离：

```
┌────────────────────────────────────────────────────┐
│                   MCP Gateway                       │
│  （统一入口：认证、限流、路由、审计）                   │
└──────────┬──────────┬──────────┬───────────────────┘
           │          │          │
    ┌──────▼───┐ ┌───▼──────┐ ┌▼───────────┐
    │ 团队A    │ │ 团队B    │ │ 团队C       │
    │ MCP      │ │ MCP      │ │ MCP         │
    │ Server   │ │ Server   │ │ Server      │
    │ ──────── │ │ ──────── │ │ ────────    │
    │ DB工具   │ │ 文档工具 │ │ 代码工具     │
    │ API工具  │ │ 搜索工具 │ │ CI/CD工具    │
    └──────────┘ └──────────┘ └────────────┘
```

设计要点：
- **统一 Gateway**：所有 MCP 请求经网关统一鉴权和路由
- **团队隔离**：每个团队的 Server 独立部署，互不影响
- **工具注册中心**：类似服务注册中心，各团队注册自己的工具
- **权限控制**：基于角色的访问控制（RBAC），不同角色可见不同工具
- **审计日志**：所有工具调用记录审计日志，满足合规要求

---

## 四、面试话术

### 话术 1："请介绍一下 MCP 协议"

> MCP 全称 Model Context Protocol，是 Anthropic 提出的一个开放、通用的应用层协议，可以理解为"AI 的 Type-C 接口"。
>
> 它解决的核心问题是工具交互的碎片化——在 MCP 之前，每个 AI 工具的接入格式都不同，集成成本极高。MCP 通过三大组件（Host/Client/Server）和标准化的 JSON-RPC 2.0 消息格式，实现了工具的"一次开发，到处接入"。
>
> 从后端视角来看，MCP 的架构思想与微服务 + API 网关非常类似：Host 类似网关负责路由和上下文管理，Client 类似 Feign Client 负责请求封装，Server 类似具体的微服务负责能力暴露。安全层面用 OAuth 2.1 with PKCE，这和我们在 Spring Security 中用的授权码模式几乎一致。
>
> 在我们的项目中，通过 LangChain4j 的 MCP 模块，将 Java 工具方法注册为 MCP Server，Agent 框架作为 Host/Client 来调度，实现了工具调用的标准化。

### 话术 2："MCP 和 Function Calling 有什么区别？"

> 这两者的区别可以用一句话概括：Function Calling 解决的是"模型怎么喊"，MCP 解决的是"工具世界怎么被组织起来"。
>
> Function Calling 是模型层面的一种能力，指的是模型能够输出结构化的 JSON 来表达"我想调用某个函数"——它是"嘴"。但 OpenAI 和 Google 的 Function Calling 格式各不相同，工具也需要硬编码在 Prompt 中。
>
> MCP 是应用层的标准化协议，它不仅规定了消息格式（JSON-RPC 2.0），还包括了工具的动态发现、上下文的统一管理、标准化的错误处理和安全认证——它是"手"，是一个完整的工程化方案。
>
> 打个比方：Function Calling 就像是你能说出"帮我查一下数据库"，MCP 则是定义了从"说出需求"到"数据库执行并返回结果"的完整通信标准和工具注册体系。

---

**相关笔记**：
- [[八股/agent/1-Agent基础概念]] — MCP 是 Agent 执行模块的标准化协议
- [[八股/agent/5-Skill技能系统]] — Skill 编排"做什么"，MCP 实现"怎么做"
- [[八股/agent/6-Memory记忆系统]] — 记忆系统通过 MCP 工具与外部存储交互
