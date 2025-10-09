# 后端同学要掌握的大模型知识 — MCP

来源：小红书笔记，作者：程序员流年

## 一、MCP 协议要解决的核心问题

### 1. 交互格式碎片化导致集成成本高
各家工具和大模型的交互格式各不相同，每个对接需要单独适配，集成代码量大且复用性差。

### 2. 上下文管理混乱交互体验差
多轮交互中缺乏统一的上下文管理机制，历史信息传递不规范，导致上下文割裂，模型响应不一致。

### 3. 版本兼容复杂维护成本高
工具侧与大模型侧版本更新频繁，缺乏统一的版本协商机制，每次升级需要手动兼容适配。

### 4. 错误处理不统一问题排查困难
不同工具的错误码和错误信息格式各异，缺乏标准化错误反馈，问题定位效率低。

### 5. 资源利用率低性能损耗大
缺乏高效的连接管理和资源调度机制，GPU 利用率约 40%，存在大量资源浪费。

---

## 二、MCP 协议定义与核心价值

MCP（Model Context Protocol）是一个开放、通用的应用层协议，专为 AI 模型与外部工具、数据源的交互而设计。可以类比为 **"AI 的 Type-C 接口"**。

### 核心价值（3 个维度）

| 维度 | 效果 |
|------|------|
| 集成效率 | 集成代码减少 80%，开发周期从 3 个月缩短至 2 周 |
| 扩展能力 | 扩展能力提升 5 倍，支持动态工具发现 |
| 资源利用率 | GPU 利用率从 40% 提升至 75% |

### 与其他技术对比

| 技术 | 关系说明 |
|------|----------|
| Prompt Engineering | Prompt 是单次交互优化，MCP 是多轮上下文管理 |
| API | API 是连接方式，MCP 是数据格式的统一标准 |
| Vector DB | 向量数据库是存储层，MCP 是传输层 |

---

## 三、MCP 架构

### 3.1 三大核心组件

#### MCP Host（宿主应用）
宿主应用的载体，Host 是 MCP 交互的"中枢节点"，核心职责：
- 管理多个 MCP Client 连接，协调不同工具、大模型的交互时序
- 负责上下文的统一管理（创建、封装、更新、过期清理），确保上下文在多组件间连贯传递
- 实现应用层业务逻辑，接收用户请求并分发至对应 MCP Client/Server，汇总处理结果后返回给用户
- 典型实现：Claude Desktop、VS Code AI 插件、企业级 AI 助手后端服务

#### MCP Client（客户端）
嵌入式接口层，部署在应用端或大模型侧，是"交互的发起者与响应解析者"，核心职责：
- 与 MCP Server 建立加密连接（支持 HTTPS、WebSocket 等传输方式）
- 将应用请求、上下文信息按照 MCP 标准格式封装，发送至 MCP Server
- 接收 MCP Server 的响应数据，按照标准格式解析后，传递给 MCP Host 或大模型
- 维护会话状态与重试逻辑，处理临时网络异常、请求超时等场景

#### MCP Server（工具服务端）
部署在外部工具、数据源侧，是"工具能力的暴露者"，核心职责：
- 暴露标准化接口，支持 3 类核心能力：
  - **资源查询**（GET 语义，如数据库查询）
  - **工具操作**（POST 语义，如发送邮件）
  - **Prompt 注入**（系统指令模板传递）
- 接收 MCP Client 的标准化请求，解析后调用本地工具能力
- 将工具执行结果按照 MCP 标准格式封装，返回给 MCP Client
- 支持协议版本协商、错误标准化反馈

### 3.2 核心交互流程

1. **上下文创建与封装**：用户发起请求后，MCP Host 收集用户需求、历史对话、场景配置等信息，按 MCP 标准格式封装上下文，生成会话 ID、角色标识、上下文内容
2. **请求标准化与发送**：MCP Client 接收 Host 传递的上下文与请求指令，封装为 MCP 标准消息（消息 ID、发送者、接收者、消息类型、内容等），通过 Stdio/HTTP+SSE/WebSocket 发送至 MCP Server
3. **请求解析与工具执行**：MCP Server 解析消息类型与内容，验证请求合法性（权限、参数格式），若验证通过则调用本地工具执行操作
4. **响应封装与回传**：工具执行完成后，MCP Server 将结果按 MCP 标准格式封装，添加标准化错误码（若执行失败），返回给 MCP Client
5. **响应解析与上下文更新**：MCP Client 解析响应数据，传递给 MCP Host；Host 根据响应结果更新上下文（添加工具返回信息、延长上下文过期时间）
6. **上下文清理与生命周期管理**

---

## 四、MCP 技术规范

### 4.1 消息结构

JSON 消息格式，核心字段：

| 字段 | 说明 |
|------|------|
| `message_id` | 消息唯一标识 |
| `sender` | 发送者标识 |
| `receiver` | 接收者标识 |
| `timestamp` | 时间戳 |
| `performative` | 消息类型（请求/响应/通知） |
| `content` | 消息内容 |
| `metadata` | 元数据 |

### 4.2 错误处理（JSON-RPC 2.0 错误码）

| 错误码 | 含义 |
|--------|------|
| -32700 | Parse error（解析错误） |
| -32600 | Invalid Request（无效请求） |
| -32601 | Method not found（方法未找到） |
| -32602 | Invalid params（无效参数） |
| -32603 | Server error（服务器错误） |

### 4.3 传输方式

| 应用场景 | 传输方式 | 核心优势 | 平均延迟 |
|----------|----------|----------|----------|
| 本地集成（如本地工具调用） | Stdio | 零网络开销，部署简单 | 1ms |
| 远程调用（如跨服务器工具对接） | HTTP+SSE | 穿透防火墙，兼容性强 | 20-100ms |
| 实时交互（如多模态实时响应） | WebSocket（开发版支持） | 双向低延迟，支持持续推送 | 5-20ms |

---

## 五、MCP 应用实例（智能数据分析助手）

### MCP 调用链

```
用户 → LLM → MCP Server → DB/API/File → LLM → 用户
```

### Step 1：MCP Server 定义工具

防止模型直接写 SQL，定义结构化安全接口：

```typescript
// tools.ts
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
]
```

### Step 2：实现 MCP Server

```typescript
import { Server } from "@modelcontextprotocol/sdk/server"
import { tools } from "./tools"
import { db } from "./db"

const server = new Server({ name: "db-mcp-server" })

server.setTools(tools)

server.onToolCall(async (tool, args) => {
  if (tool === "list_tables") {
    return await db.listTables()
  }
  if (tool === "query_table") {
    return await db.select(args.table, args.columns, args.limit)
  }
})

server.start()
```

### Step 3：LLM 如何使用 MCP

用户输入 "分析一下最近 30 天用户增长趋势"，LLM 内部自动完成（非手工编码）：
- 识别用户意图 → 选择合适的 MCP 工具 → 调用 `list_tables` 查看表 → 调用 `query_table` 查询数据 → 分析结果并返回给用户

---

## MCP 注意要点

### 安全规范
- 传输安全：HTTPS/TLS 1.3+
- 认证授权：OAuth 2.1 with PKCE
- 凭证管理：安全存储与轮换
- 审计日志：完整操作记录

### 性能优化
- 上下文分片（Context Sharding）
- 连接复用
- 按需调用
- 批量处理
