---
tags:
  - 八股
  - AI
  - Agent
status: 待复习
created: 2026-05-16
---

# AI Agent 基础概念

> AI Agent 面试八股第一模块：从定义到工程实现的完整知识体系。Agent 是当前大模型应用落地的核心范式，也是面试高频考点。

---

## 一、知识点讲解

### 1. AI Agent 定义

AI Agent（智能体）是一种能够**自主感知环境、制定计划、调用工具、执行动作**的智能系统。它不是简单的"LLM + Prompt"，而是一个以大语言模型为核心大脑、具备**感知-规划-执行-反思**闭环能力的完整工程系统。

核心特征拆解：

| 特征 | 含义 | 类比 |
|------|------|------|
| **自主性(Autonomy)** | 无需人类逐步指令，能独立完成任务分解与执行 | 像一个能独立工作的员工 |
| **感知性(Perception)** | 能接收外部环境信息（用户输入、API返回、传感器数据） | 像人的五感 |
| **反应性(Reactivity)** | 能根据环境变化动态调整行为 | 像人遇到突发情况改变计划 |
| **主动性(Proactivity)** | 能主动发起行动，不仅仅被动响应 | 像主动发现问题的管理者 |
| **社交性(Social)** | 能与其他 Agent 或人类协作 | 像团队协作中的沟通能力 |

一句话定义：**Agent = LLM + 记忆 + 规划 + 工具使用 + 自主决策循环**。

---

### 2. Agent vs 传统 LLM 对比

| 维度 | 传统 LLM（如 ChatGPT 裸对话） | AI Agent |
|------|------|------|
| **交互模式** | 单轮/多轮对话，一问一答 | 自主多步执行，最小化人工干预 |
| **记忆能力** | 仅上下文窗口内的对话历史 | 短期记忆（上下文）+ 长期记忆（向量库/数据库） |
| **工具使用** | 无，或仅内置插件 | 可动态发现、选择、调用外部工具（API/数据库/代码执行） |
| **推理方式** | 单次推理，输出即结果 | 循环推理：思考→行动→观察→再思考（ReAct 循环） |
| **纠错能力** | 无，出错需人工纠正 | 具备自我反思和重试机制，能根据反馈调整策略 |
| **任务范围** | 只能处理文本生成类任务 | 能完成跨系统的复杂工作流（如订票+发邮件+更新数据库） |
| **主动性** | 被动响应 | 主动规划、自主决策 |

**本质区别**：传统 LLM 是"你问我答"的工具，Agent 是"你说目标，我来搞定"的助手。核心差异在于 **Agent 具备自主决策循环**，而不是单次推理就结束。

---

### 3. Agent 七大核心组件

```
┌──────────────────────────────────────────────────────┐
│                    AI Agent 架构                       │
│                                                       │
│  ┌──────────┐    ┌──────────────┐    ┌──────────┐    │
│  │ ① 感知模块 │───▶│ ② 核心大脑   │───▶│ ③ 规划模块 │   │
│  │(Perception)│    │   (LLM)     │    │(Planning) │   │
│  └──────────┘    └──────┬───────┘    └─────┬────┘    │
│       ▲                │                    │          │
│       │                ▼                    ▼          │
│  ┌────┴─────┐   ┌──────────┐       ┌──────────┐      │
│  │ ⑥ 反馈闭环 │◀──│ ⑤ 执行模块 │◀──────│ ④ 记忆模块 │      │
│  │(Feedback) │   │(Execution)│       │(Memory)  │      │
│  └──────────┘   └──────────┘       └──────────┘      │
│       │                                               │
│       ▼                                               │
│  ┌──────────┐                                         │
│  │ ⑦ 调度模块 │   负责协调各组件间的流转与优先级         │
│  │(Scheduler)│                                         │
│  └──────────┘                                         │
└──────────────────────────────────────────────────────┘
```

| # | 组件 | 职责 | 技术实现 |
|---|------|------|---------|
| ① | **感知模块** | 接收用户输入、环境信号、工具返回结果 | 用户输入解析、API 响应解析、事件监听 |
| ② | **核心大脑(LLM)** | 理解意图、生成推理、决策下一步动作 | GPT-4、Claude、Qwen 等大模型 |
| ③ | **规划模块** | 将复杂任务分解为子步骤，制定执行计划 | ReAct、Plan-and-Solve、ToT |
| ④ | **记忆模块** | 存储历史交互、学习经验知识 | 短期：上下文窗口；长期：向量数据库 |
| ⑤ | **执行模块** | 调用工具/API，执行具体操作 | Function Calling、MCP 协议 |
| ⑥ | **反馈闭环** | 评估执行结果，决定是否需要重试或调整 | 结果校验、反思评分、异常处理 |
| ⑦ | **调度模块** | 协调各组件流转，管理并发与优先级 | 状态机、DAG 工作流、消息队列 |

---

### 4. Agent 整体流程

Agent 的核心运转遵循 **感知 → 规划 → 执行 → 反思记忆** 的闭环：

```
用户目标输入
     │
     ▼
┌──────────┐
│  感知阶段  │  接收用户意图，解析任务上下文
└────┬─────┘
     │
     ▼
┌──────────┐
│  规划阶段  │  LLM 推理 + 任务分解 → 生成行动计划
└────┬─────┘
     │
     ▼
┌──────────┐     ┌──────────┐
│  执行阶段  │────▶│ 工具调用   │  Function Calling / MCP
└────┬─────┘     └────┬─────┘
     │                │
     ▼                │
┌──────────┐          │
│ 反思阶段  │◀─────────┘  观察执行结果，判断是否达标
└────┬─────┘
     │
     ├── 成功 → 返回最终结果
     │
     └── 失败/未完成 → 回到【规划阶段】重新制定策略（带反思信息）
```

关键点：
- 这是一个**循环**而非线性流程，Agent 会反复执行直到任务完成或达到终止条件
- 每一轮循环中，**记忆**都在持续更新（短期上下文累积 + 长期经验沉淀）
- **反思**环节是 Agent 区别于简单工具调用的核心——它能让 Agent 从错误中学习

---

### 5. Agent 工程实现全景

从工程实现角度，构建一个 Agent 需要六大模块：

#### 5.1 感知层

```python
# 用户输入解析示例
user_input = "帮我查一下北京明天的天气，如果下雨就帮我发邮件提醒团队带伞"
# Agent 需要感知：
#   1. 意图：查天气 + 条件判断 + 发邮件
#   2. 实体：北京、明天、团队
#   3. 条件：如果下雨
```

#### 5.2 规划层（三种主流方案）

| 方案 | 思路 | 适用场景 |
|------|------|---------|
| **ReAct** | 思考-行动-观察交替进行，逐步推进 | 通用场景，最常用 |
| **Planner** | 一次性生成完整计划，再逐步执行 | 步骤明确的复杂任务 |
| **状态机(FSM)** | 预定义状态转移规则，LLM 只做节点决策 | 流程固定的业务场景 |

#### 5.3 行动层

- **Function Calling**：OpenAI 提出的标准，LLM 输出结构化 JSON 描述要调用的工具
- **MCP（Model Context Protocol）**：Anthropic 提出的开放协议，标准化工具发现与调用 → 详见 [[八股/agent/MCP]]

#### 5.4 记忆层

| 类型 | 实现 | 特点 |
|------|------|------|
| 短期记忆 | 对话上下文（Context Window） | 容量有限，随会话消失 |
| 长期记忆 | 向量数据库（Milvus/Chroma/Pinecone） | 持久化，可跨会话检索 |
| 工作记忆 | Scratchpad / 中间变量 | 当前任务执行过程中的临时信息 |

→ 记忆系统详见 [[八股/agent/Memory]]

#### 5.5 控制流（安全兜底）

```yaml
# Agent 安全控制配置示例
safety:
  max_iterations: 10        # 最大循环轮次，防止死循环
  timeout_per_step: 30s     # 单步超时
  total_timeout: 300s       # 总超时
  max_tokens_per_call: 4096 # 单次 LLM 调用 token 上限
  allowed_tools:            # 白名单工具
    - weather_api
    - email_sender
    - calculator
  forbidden_tools:          # 黑名单
    - file_delete
    - shell_execute
  permission_level: readonly # 只读 / 读写 / 管理员
```

#### 5.6 编排层

- **单 Agent**：DAG 工作流（有向无环图），定义子任务间的依赖关系
- **多 Agent**：引入协作编排，详见 [[八股/agent/Multi-Agent多智能体系统]]

---

### 6. Agent 框架选型

| 框架 | 定位 | 核心特点 | 适用场景 | 学习曲线 |
|------|------|---------|---------|---------|
| **LangChain** | Agent 开发工具链 | 模块化组件（Chain/Tool/Memory），生态最丰富 | 快速原型、通用 Agent 开发 | 中等 |
| **LlamaIndex** | RAG + Agent 框架 | 强大的数据索引与检索能力，RAG 场景最优 | 知识库问答、文档 Agent | 中等 |
| **MetaGPT** | 多智能体协作 | 角色扮演（PM/架构师/工程师），SOP 驱动 | 软件开发自动化 | 较高 |
| **LangGraph** | Agent 编排引擎 | 基于图的状态机，精确控制 Agent 流转 | 复杂工作流、需要精细控制流转 | 较高 |
| **AutoGen** | 多 Agent 对话框架 | Agent 间自动对话协作，微软出品 | 多 Agent 协作研究 | 中等 |
| **CrewAI** | 多 Agent 协作框架 | 角色化 Agent + 任务委派，API 简洁 | 业务流程自动化 | 较低 |

选型建议：
- **入门/通用**：LangChain（生态最全，资料最多）
- **RAG 场景**：LlamaIndex
- **需要精确控制流程**：LangGraph
- **多 Agent 协作**：CrewAI（简单）或 MetaGPT（专业）
- **研究/实验**：AutoGen

---

### 7. Function Calling 核心逻辑

Function Calling 是 Agent 调用外部工具的核心机制，流程如下：

```
┌─────────────────────────────────────────────────────────────┐
│                    Function Calling 流程                      │
│                                                              │
│  ① 注册工具 Schema                                           │
│  ┌───────────────────────────────────────┐                   │
│  │ tools: [{                              │                   │
│  │   name: "get_weather",                │                   │
│  │   description: "查询城市天气",          │                   │
│  │   parameters: {                       │                   │
│  │     type: "object",                   │                   │
│  │     properties: {                     │                   │
│  │       city: { type: "string" }        │                   │
│  │     }                                 │                   │
│  │   }                                   │                   │
│  │ }]                                    │                   │
│  └─────────────────┬─────────────────────┘                   │
│                    │                                          │
│  ② LLM 语义匹配    ▼                                         │
│  ┌───────────────────────────────────────┐                   │
│  │ 用户: "北京天气怎么样"                   │                   │
│  │ LLM 推理: 需要 get_weather(city="北京") │                  │
│  └─────────────────┬─────────────────────┘                   │
│                    │                                          │
│  ③ 输出结构化 JSON  ▼                                         │
│  ┌───────────────────────────────────────┐                   │
│  │ { "tool_calls": [{                    │                   │
│  │     "function": {                     │                   │
│  │       "name": "get_weather",          │                   │
│  │       "arguments": "{ \"city\": \"北京\" }"                │
│  │     }                                 │                   │
│  │ }] }                                  │                   │
│  └─────────────────┬─────────────────────┘                   │
│                    │                                          │
│  ④ 工程侧解析执行  ▼                                         │
│  ┌───────────────────────────────────────┐                   │
│  │ 解析 JSON → 调用 get_weather("北京")    │                  │
│  │ 得到结果: {"temp": "25°C", "weather": "晴"}               │
│  └─────────────────┬─────────────────────┘                   │
│                    │                                          │
│  ⑤ 结果回传 LLM    ▼                                         │
│  ┌───────────────────────────────────────┐                   │
│  │ 将工具结果作为新消息追加到上下文          │                   │
│  │ LLM 基于结果生成最终回答                 │                   │
│  │ "北京今天25°C，天气晴朗"                │                   │
│  └───────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

关键代码示例（OpenAI 格式）：

```python
# ① 定义工具 Schema
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "查询指定城市的天气信息",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，如'北京'、'上海'"
                }
            },
            "required": ["city"]
        }
    }
}]

# ② 发送给 LLM（带工具定义）
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "北京天气怎么样"}],
    tools=tools
)

# ③ LLM 返回工具调用意图（结构化 JSON）
tool_call = response.choices[0].message.tool_calls[0]
# tool_call.function.name = "get_weather"
# tool_call.function.arguments = '{"city": "北京"}'

# ④ 工程侧解析并执行
args = json.loads(tool_call.function.arguments)
result = get_weather(args["city"])

# ⑤ 结果回传 LLM
messages.append(response.choices[0].message)
messages.append({
    "role": "tool",
    "tool_call_id": tool_call.id,
    "content": json.dumps(result)
})
final_response = client.chat.completions.create(
    model="gpt-4",
    messages=messages,
    tools=tools
)
```

---

### 8. Function Calling vs Toolformer

| 维度 | Function Calling | Toolformer |
|------|-----------------|------------|
| **工作阶段** | **推理阶段（Inference-time）** | **训练阶段（Training-time）** |
| **核心思路** | 在 Prompt 中提供工具 Schema，LLM 在推理时决定是否调用 | 在预训练/SFT 阶段就教会模型何时调用工具 |
| **工具范围** | 动态注册，运行时可增减工具 | 固定在模型权重中，灵活性低 |
| **额外训练** | 不需要，开箱即用 | 需要专门的工具调用数据集做微调 |
| **适用模型** | GPT-4、Claude 等主流模型均支持 | Meta 提出，需要专门训练的模型 |
| **灵活性** | 高，可随时更换工具 | 低，工具绑定在模型中 |
| **精度** | 依赖 Prompt 工程，可能选错工具 | 训练过，工具选择更精准 |

一句话总结：**Function Calling 是"考试时给参考书"（推理时提供工具），Toolformer 是"提前学会用工具"（训练时学会调用）**。

---

### 9. 规划能力方法

| 方法 | 结构 | 思路 | 复杂度 | 适用场景 |
|------|------|------|--------|---------|
| **CoT（Chain-of-Thought）** | 线性链 `A→B→C→D` | 逐步推理，每步依赖上一步 | O(n) | 简单线性推理 |
| **ToT（Tree-of-Thought）** | 树状结构 | 每步生成多个候选方案，搜索最优路径 | O(b^d) | 需要探索多种可能 |
| **GoT（Graph-of-Thought）** | 图结构 | 推理步骤可合并/回溯/并行 | 取决于图密度 | 复杂推理，步骤间有依赖 |
| **Plan-and-Solve** | 先整体后局部 | 先生成完整计划，再逐步执行 | O(n) | 步骤明确的复杂任务 |
| **分层规划（HWP）** | 层次化 | 高层制定粗略计划，低层细化执行 | O(log n) | 大型多阶段任务 |

```
CoT（线性）:        ToT（树状）:         GoT（图状）:
 A → B → C → D        A                     A ──┐
                       ├─ B1 ── C1          │    ▼
                       ├─ B2 ── C2          B ←→ C
                       └─ B3                │    ▼
                                           D ←──┘
```

与数据结构的对应关系：
- CoT → 链表（线性遍历）
- ToT → 树（DFS/BFS 搜索）
- GoT → 有向图（拓扑排序、可达性分析）

---

### 10. Agent 安全性

#### 10.1 权限沙箱

```python
# Agent 执行环境隔离
class AgentSandbox:
    def __init__(self):
        self.allowed_apis = {"weather_api", "email_sender"}
        self.max_file_size = 10 * 1024 * 1024  # 10MB
        self.network_whitelist = ["api.weather.com", "smtp.company.com"]
        self.readonly_paths = ["/data/public"]
```

#### 10.2 四层安全防护

| 层级 | 措施 | 说明 |
|------|------|------|
| **工具层** | 白名单 + 参数校验 | 只允许调用预注册工具，参数类型和范围校验 |
| **执行层** | 沙箱隔离 + 资源限制 | 容器化执行，限制 CPU/内存/网络/磁盘 |
| **数据层** | 脱敏 + 加密 | 敏感数据传输加密，日志脱敏 |
| **审计层** | 全链路日志 + 人工审批 | 记录所有工具调用，高风险操作需人工确认 |

#### 10.3 防止越权操作

- **最小权限原则**：Agent 默认只有只读权限，写操作需显式授权
- **审批机制**：高风险操作（删除数据、转账等）触发人工审批流
- **Prompt 注入防御**：对用户输入做清洗，区分系统指令和用户数据

#### 10.4 工具调用失效处理

```python
def safe_tool_call(tool, args, max_retries=3):
    for attempt in range(max_retries):
        try:
            result = tool.execute(args)
            if result.is_valid():
                return result
            else:
                # 结果不合法，让 LLM 反思调整
                reflect_and_retry(result.error)
        except TimeoutError:
            log.warning(f"工具 {tool.name} 超时，第 {attempt+1} 次重试")
        except APIError as e:
            if e.status_code == 429:  # 限流
                wait_and_retry(backoff=2**attempt)
            elif e.status_code >= 500:  # 服务端错误
                switch_to_backup_tool(tool)
            else:
                raise
    # 所有重试失败，降级处理
    return fallback_response("工具暂时不可用，已记录并通知管理员")
```

---

### 11. 输出稳定性：保证大模型输出稳定 JSON

大模型输出 JSON 不稳定是 Agent 工程中的常见痛点，解决方法：

| 方法 | 原理 | 示例 |
|------|------|------|
| **Schema 指令** | 在 System Prompt 中明确指定 JSON Schema | `"你必须严格按照以下 JSON Schema 输出..."` |
| **固定模板** | 给出完整的输出模板，LLM 只需填空 | `"请按以下格式回答：{\"result\": \"...\", \"confidence\": 0.0-1.0}"` |
| **分隔符约束** | 用特殊标记包裹 JSON 部分 | `"将 JSON 结果放在 <json>...</json> 标签之间"` |
| **格式校验 + 重试** | 解析失败则将错误信息回传 LLM 重试 | `try: json.loads(text) except: retry_with_error_message()` |
| **结构化输出 API** | 使用模型原生支持的 Structured Output | OpenAI 的 `response_format={"type": "json_object"}` |
| **JSON Mode** | 强制模型输出合法 JSON | 部分模型提供 JSON Mode 开关 |
| **Pydantic 校验** | 工程侧用 Pydantic 做二次校验 | `result = WeatherResponse.model_validate_json(raw_output)` |

```python
# 综合方案：Prompt 约束 + 格式校验 + 自动重试
async def structured_llm_call(prompt, schema, max_retries=3):
    system_prompt = f"""你是一个JSON输出助手。
你必须严格按照以下Schema输出，不要输出任何JSON以外的内容：
{json.dumps(schema, ensure_ascii=False, indent=2)}
将你的JSON回答放在<json>和</json>标签之间。"""

    for attempt in range(max_retries):
        raw = await llm.chat(system=system_prompt, user=prompt)
        try:
            # 提取 JSON
            json_str = extract_between_tags(raw, "json")
            result = json.loads(json_str)
            # Pydantic 校验
            return schema_model.model_validate(result)
        except (json.JSONDecodeError, ValidationError) as e:
            if attempt < max_retries - 1:
                # 把错误信息喂给 LLM 让它修正
                prompt += f"\n\n上次输出格式有误：{str(e)}，请修正后重新输出。"
            else:
                raise FormatError(f"重试{max_retries}次后仍无法获得合法JSON")
```

---

## 二、串联理解记忆

### 知识模块关联图

```
                    ┌──────────────────────┐
                    │   1-Agent基础概念     │ ◀── 你在这里
                    │  （定义/组件/流程）    │
                    └──────────┬───────────┘
                               │
         ┌─────────────┬───────┼────────┬──────────────┐
         │             │       │        │              │
         ▼             ▼       ▼        ▼              ▼
  ┌──────────┐  ┌──────────┐ ┌────┐ ┌────────┐  ┌──────────┐
  │  ReAct   │  │  MCP     │ │RAG │ │ Memory │  │ Java后端  │
  │ 推理范式  │  │ 工具协议  │ │检索 │ │ 记忆   │  │ 基础设施  │
  └──────────┘  └──────────┘ └────┘ └────────┘  └──────────┘
```

### 关联解读

**1. Agent 基础 → ReAct 推理范式**
Agent 的核心运转流程"感知→规划→执行→反思"本质上就是一个 ReAct 循环（Reason + Act）。ReAct 给出了 Agent 推理模式的具体实现方案——交替生成"思考"和"行动"。理解了 Agent 的闭环流程，再去看 [[八股/agent/ReAct]] 就会知道 ReAct 是"怎么想的"，Agent 架构是"怎么搭的"。

**2. Agent 基础 → MCP 协议**
Agent 需要调用工具，但各家模型和工具的接口格式不统一。MCP（Model Context Protocol）就是为了解决这个问题——它是工具调用的"Type-C 接口"，标准化了工具发现、调用和结果返回的流程。[[八股/agent/MCP]] 是 Agent 执行层的工程标准化方案。

**3. Agent 基础 → Memory 记忆系统**
Agent 七大组件中，记忆模块是贯穿所有环节的核心。没有记忆，Agent 就像一个失忆的助手——每次都从头开始。短期记忆负责当前任务的上下文传递，长期记忆负责跨会话的知识积累。[[八股/agent/Memory]] 详解了如何构建多层次的记忆体系。

**4. Agent 基础 → RAG**
Agent 的一个核心能力是"知识检索"——当 LLM 自身知识不够时，需要从外部知识库中检索相关信息。RAG（Retrieval-Augmented Generation）正是这个能力的底层技术。[[八股/agent/RAG]] 是 Agent 感知和记忆能力的重要补充。

**5. Agent 基础 → Java 后端**
Agent 的工程实现大量依赖后端基础设施：
- **Redis**：短期记忆缓存、分布式锁、工具调用限流
- **MySQL/PostgreSQL**：对话历史持久化、工具注册表、审计日志
- **Kafka/消息队列**：Agent 异步任务编排、多 Agent 间通信
- **微服务架构**：Agent 编排服务、工具执行服务、记忆服务等独立部署
- **向量数据库**（Milvus/Chroma）：长期记忆存储与检索

**6. CoT/ToT/GoT 与算法中树/图结构的对应**

| 推理方法 | 数据结构 | 算法思想 |
|---------|---------|---------|
| CoT（链式推理） | 链表（Linked List） | 线性遍历，O(n) |
| ToT（树状推理） | N叉树（N-ary Tree） | DFS/BFS 搜索最优路径 |
| GoT（图状推理） | 有向图（DAG） | 拓扑排序、多路径汇聚 |

理解了这个对应关系，就能用算法思维来理解 Agent 的规划策略——CoT 就是贪心（只走一条路），ToT 就是回溯搜索（多条路试），GoT 就是动态规划（子问题复用）。

---

### 记忆口诀

> **"感规执反七组件，智能体比LLM多一圈；**
> **Function Calling 五步走，Schema匹配JSON传；**
> **CoT线性ToT树，GoT图状规划全；**
> **沙箱权限加脱敏，Agent安全四道关。"**

解读：
- **感规执反七组件**：感知、规划、执行、反思，加上 LLM 大脑、记忆、调度共七大组件
- **多一圈**：Agent 比 LLM 多了一个自主决策循环
- **五步走**：注册Schema → 语义匹配 → 输出JSON → 解析执行 → 结果回传
- **四道关**：工具层、执行层、数据层、审计层四层安全防护

---

## 三、面试题

### Q1：LLM Agent 的定义是什么？和"LLM+工具调用"有什么本质区别？

**答：**

LLM Agent 是一种以大语言模型为核心决策引擎、能够**自主感知环境、规划任务、调用工具、执行动作并反思调整**的智能系统。

和"LLM+工具调用"的本质区别在于**自主性**：
- "LLM+工具调用"是被动的：用户指定调什么工具，LLM 只是执行
- Agent 是主动的：用户给一个目标，Agent 自主决定调什么工具、按什么顺序调、失败了怎么办
- Agent 具备**闭环反馈**能力——能观察执行结果并据此调整后续行为
- Agent 具备**记忆**能力——能跨轮次保留上下文和经验

核心公式：**Agent = LLM + 记忆 + 规划 + 工具使用 + 自主决策循环**

---

### Q2：Agent 整体流程是怎么做的？包括哪些模块？

**答：**

Agent 的核心流程是 **感知 → 规划 → 执行 → 反思记忆** 的闭环：

1. **感知**：接收用户输入、解析意图和实体
2. **规划**：LLM 进行推理，将任务分解为可执行的子步骤（使用 ReAct/CoT 等方法）
3. **执行**：通过 Function Calling 或 MCP 调用外部工具完成具体操作
4. **反思**：观察执行结果，判断是否达标，不达标则更新策略重新规划

核心模块包括七大组件：感知模块、核心大脑（LLM）、规划模块、记忆模块（短期上下文 + 长期向量库）、执行模块、反馈闭环、调度模块。

这是一个**循环**过程，Agent 会反复执行直到任务完成或达到终止条件（最大轮次/超时等安全兜底）。

---

### Q3：怎么实现智能体（Agent）？

**答：**

从工程实现角度，构建一个 Agent 需要六大模块：

1. **感知层**：解析用户输入，提取意图和实体
2. **规划层**：用 ReAct（交替思考行动）、Planner（一次性规划）、或状态机（预定义流程）来制定执行计划
3. **行动层**：通过 Function Calling（给 LLM 提供工具 Schema，LLM 输出结构化 JSON 描述调用意图）或 MCP 协议实现工具调用
4. **记忆层**：短期记忆用上下文窗口管理，长期记忆用向量数据库（Milvus/Chroma）持久化
5. **控制流**：设置执行上限（最大轮次）、超时机制、权限白名单等安全兜底
6. **编排层**：用 DAG 工作流定义子任务依赖关系，多 Agent 场景下引入协作编排

技术选型上，可以用 LangChain/LangGraph 快速搭建，也可以用 Spring Boot + Redis + 向量库从零实现。

---

### Q4：目前有哪些主流方法可以赋予 LLM 规划能力？

**答：**

主要有五种方法：

| 方法 | 思路 | 特点 |
|------|------|------|
| **CoT（Chain-of-Thought）** | 线性逐步推理，每步依赖上一步 | 简单高效，适合线性任务 |
| **ToT（Tree-of-Thought）** | 每步生成多个候选，构建搜索树探索最优路径 | 能探索更多可能，但成本高 |
| **GoT（Graph-of-Thought）** | 推理步骤构成图结构，支持合并、回溯、并行 | 最灵活，能处理复杂依赖关系 |
| **Plan-and-Solve** | 先生成完整计划再逐步执行 | 适合步骤明确的复杂任务 |
| **分层规划（Hierarchical Planning）** | 高层制定粗略计划，低层细化执行 | 适合大型多阶段任务 |

实际应用中，**ReAct（Reason+Act）** 是最常用的范式，它将 CoT 的推理能力和工具执行结合起来，形成"思考→行动→观察"的循环。

---

### Q5：Agent 的工具调用（Function Calling）核心逻辑是什么？

**答：**

Function Calling 的核心流程分五步：

1. **注册工具 Schema**：在请求中告诉 LLM 有哪些可用工具，每个工具的名称、描述、参数 Schema
2. **LLM 语义匹配**：LLM 根据用户意图和工具描述，判断是否需要调用工具、调用哪个工具
3. **输出结构化 JSON**：如果需要调用，LLM 输出一个结构化的 JSON，包含工具名称和参数
4. **工程侧解析执行**：解析 JSON，在工程侧实际调用对应的函数/API
5. **结果回传 LLM**：将工具执行结果追加到上下文中，让 LLM 基于结果生成最终回答

关键点：**LLM 本身不执行工具**，它只负责"决策"——决定调什么、传什么参数。实际的工具执行发生在工程侧。

---

### Q6：Function Calling 和 Toolformer 本质区别？

**答：**

本质区别在于**工作阶段**：

- **Function Calling** 是**推理阶段**的方案：运行时通过 Prompt 提供工具 Schema，LLM 在推理时动态决定调用哪个工具。不需要额外训练，灵活性强，工具可随时增减。

- **Toolformer** 是**训练阶段**的方案：在预训练/SFT 阶段就用包含工具调用的数据集训练模型，让模型"学会"何时调用工具。调用能力固化在模型权重中，精度更高但灵活性低。

比喻：Function Calling 是"考试时给参考书"（运行时查工具手册），Toolformer 是"提前学会用工具"（训练时就学会）。

---

### Q7：大模型工具调用如何避免参数格式错误？

**答：**

多层防护策略：

1. **Prompt 层**：在 System Prompt 中明确描述参数类型、格式要求、枚举值
2. **Schema 层**：使用 JSON Schema 严格定义参数结构（类型、必填、取值范围）
3. **结构化输出**：使用模型的 Structured Output / JSON Mode 强制输出合法 JSON
4. **分隔符**：用 `<json>...</json>` 等标签包裹输出，便于准确提取
5. **工程侧校验**：用 Pydantic/Zod 做 Schema 校验，类型不匹配则拦截
6. **错误重试**：校验失败时将错误信息回传 LLM，让它修正后重新输出
7. **兜底策略**：多次重试失败后，使用默认值或降级处理

---

### Q8：多个工具可完成任务时 Agent 如何选择？

**答：**

LLM 选择工具的核心依据是**语义匹配**——比较用户意图和每个工具的 description。具体机制：

1. **工具描述质量**：description 写得越精确，LLM 越容易选对。好的 description 应包含：功能说明、适用场景、输入输出说明
2. **Few-shot 示例**：在 Prompt 中给出"什么情况用什么工具"的示例，引导 LLM 正确选择
3. **工具数量控制**：可用工具太多时，先做一轮预筛选（基于关键词/向量相似度），只把最相关的 Top-K 工具传给 LLM
4. **路由分类器**：用一个轻量模型先做意图分类，再根据分类结果只传对应类别的工具给 LLM
5. **多工具并行调用**：如果多个工具都需要，可以并行调用（如同时查天气和查航班）

---

### Q9：Agent 怎么评估效果？

**答：**

Agent 评估是多维度的：

| 维度 | 指标 | 说明 |
|------|------|------|
| **任务完成率** | Task Success Rate | 最终是否正确完成了用户任务 |
| **步骤效率** | Step Efficiency | 完成任务用了多少步，越少越好 |
| **工具使用准确性** | Tool Selection Accuracy | 选对工具的比例 |
| **成本效率** | Token Efficiency | 总消耗的 token 数 |
| **延迟** | End-to-End Latency | 从用户输入到最终输出的时间 |
| **鲁棒性** | Robustness Score | 面对异常输入/工具失败时的处理能力 |

评估方法：
- **人工评估**：人工标注任务完成质量（最准确但成本高）
- **LLM-as-Judge**：用另一个 LLM 来评判 Agent 的输出质量
- **自动化测试集**：预定义"任务+期望结果"的测试用例，跑批评估

---

### Q10：Agent 的鲁棒性如何测试？

**答：**

鲁棒性测试的核心思路是**模拟各种异常场景**，验证 Agent 的容错能力：

1. **工具失败场景**：模拟 API 超时、返回 500 错误、空结果，看 Agent 是否能优雅降级
2. **参数异常场景**：传入非法参数、缺失参数、超范围值，看 Agent 是否能正确处理
3. **Prompt 注入场景**：用户输入中嵌入恶意指令，验证安全防线是否生效
4. **边界条件**：极端长输入、循环依赖任务、不存在的工具需求
5. **多轮一致性**：长对话中上下文是否一致，是否会出现遗忘
6. **压力测试**：高并发调用下的稳定性和性能

测试方法：**对抗测试（Adversarial Testing）**——故意构造各种"刁难"场景，确保 Agent 不会崩溃或做出危险操作。

---

### Q11：如何防止 Agent 隐私泄露与越权操作？

**答：**

四层防护体系：

1. **工具层——白名单 + 参数校验**：只暴露必要的工具，参数类型和范围严格校验
2. **执行层——沙箱隔离**：Agent 在容器中运行，限制网络访问、文件读写、系统调用
3. **数据层——脱敏 + 加密**：
   - 用户输入送入 LLM 前做敏感信息脱敏（手机号、身份证号等替换为占位符）
   - 工具调用结果中的敏感字段加密存储
   - 日志中不记录完整敏感数据
4. **审计层——全链路日志 + 人工审批**：
   - 记录 Agent 的每一次工具调用（调用什么、传了什么参数、返回什么结果）
   - 高风险操作（删除数据、转账等）触发人工审批流程
   - 定期审计日志，发现异常行为

另外还需要做 **Prompt 注入防御**：对用户输入做清洗，区分系统指令和用户数据，防止用户通过构造输入来"劫持"Agent 的行为。

---

### Q12：Agent 如何处理工具调用失效？

**答：**

工具调用失效的处理策略分三级：

1. **自动重试**：
   - 超时/网络错误：指数退避重试（2^attempt 秒后重试）
   - 限流(429)：等待后重试
   - 最多重试 3 次，避免无限循环

2. **降级方案**：
   - 主工具失败时切换到备选工具（如主天气 API 挂了，切到备用天气 API）
   - 没有备选工具时，告知用户并给出替代建议

3. **LLM 反思调整**：
   - 将错误信息回传给 LLM："工具 X 返回了错误 Y，请调整策略"
   - LLM 可能会选择换一个工具，或者修改参数重试
   - 这是 Agent 比普通工具调用更强大的地方——它能从错误中学习

```python
# 典型的失效处理伪代码
for attempt in range(max_retries):
    result = tool.execute(args)
    if result.success:
        return result.data
    elif result.is_retryable:
        wait(backoff=2**attempt)
        continue
    else:
        # 不可重试的错误，让 LLM 反思
        llm_reflect(result.error)  # LLM 调整策略
        break
```

---

### Q13：Agent 调用工具不正确怎么办？SFT 和 RL 如何解决？

**答：**

Agent 调用工具不正确（选错工具、参数错误、调用顺序错误）是常见问题，可通过两种方式优化：

**SFT（监督微调）**：
- 收集"用户意图→正确工具调用"的标注数据
- 用这些数据微调模型，让模型学会在什么场景下应该调什么工具
- 优点：见效快，能快速修正常见错误模式
- 缺点：需要大量标注数据，覆盖面有限

**RL（强化学习）**：
- 设计奖励函数：工具调用正确给正奖励，错误给负奖励
- 使用 RLHF 或 RLAIF 方法，让模型通过试错学习最优工具调用策略
- 优点：能发现更优策略，不依赖人工标注
- 缺点：训练成本高，奖励函数设计困难

**工程层面的补充**：
- 在 Prompt 中增加 Few-shot 示例（低成本即时生效）
- 对工具描述(description)做优化，让 LLM 更容易区分工具
- 加一层规则校验，在调用前拦截明显的错误

---

### Q14：如何保证大模型输出稳定 JSON？

**答：**

综合方案，从 Prompt 到工程校验多层保障：

1. **Prompt 约束**：
   - 在 System Prompt 中明确指定 JSON Schema
   - 给出完整的输出模板（LLM 只需填空）
   - 用 `<json>...</json>` 标签包裹 JSON 部分

2. **模型能力**：
   - 使用 Structured Output API（如 OpenAI 的 `response_format`）
   - 开启 JSON Mode 强制输出合法 JSON
   - 选择支持 JSON 输出更好的模型版本

3. **工程校验**：
   - `json.loads()` 解析后用 Pydantic 做 Schema 校验
   - 解析失败则将错误信息回传 LLM 重试（最多 3 次）
   - 所有重试失败后走兜底逻辑（默认值 / 人工介入）

4. **最佳实践**：
   - JSON 结构尽量扁平，嵌套不超过 3 层
   - 枚举值用 `enum` 约束，数值范围用 `minimum/maximum` 约束
   - 给出正例和反例，让 LLM 理解什么是正确的输出

---

### Q15：了解哪些 Agent 框架？怎么选型？

**答：**

主流框架及其定位：

- **LangChain**：最全面的 Agent 开发工具链，模块化设计（Chain/Tool/Memory/Agent），生态最丰富，适合快速原型和通用 Agent 开发
- **LangGraph**：LangChain 团队出品，基于图的状态机引擎，适合需要精细控制 Agent 流转的复杂工作流
- **LlamaIndex**：专注 RAG 场景，数据索引和检索能力最强，适合知识库问答类 Agent
- **MetaGPT**：角色扮演驱动的多 Agent 框架（PM/架构师/工程师），适合软件开发自动化
- **AutoGen**：微软出品，多 Agent 自动对话协作，适合研究场景
- **CrewAI**：角色化 Agent + 任务委派，API 简洁上手快，适合业务流程自动化

选型建议：
- **入门/通用需求** → LangChain
- **RAG/知识库场景** → LlamaIndex
- **需要精确控制流程** → LangGraph
- **多 Agent 协作/业务自动化** → CrewAI
- **软件开发场景** → MetaGPT

---

### Q16：如何实现使用大模型调自己的接口？

**答：**

核心就是实现 Function Calling 的工具注册和执行循环：

```python
# 第一步：定义你的接口为工具 Schema
tools = [{
    "type": "function",
    "function": {
        "name": "query_user_info",
        "description": "查询用户基本信息",
        "parameters": {
            "type": "object",
            "properties": {
                "user_id": {"type": "string", "description": "用户ID"},
                "fields": {"type": "array", "items": {"type": "string"},
                           "description": "需要查询的字段列表"}
            },
            "required": ["user_id"]
        }
    }
}]

# 第二步：维护工具名 → 实际函数的映射
tool_map = {
    "query_user_info": my_user_service.get_user_info,
    "send_notification": my_notify_service.send,
}

# 第三步：实现调用循环
def agent_run(user_message):
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = llm.chat(messages=messages, tools=tools)

        if not response.tool_calls:  # LLM 没有要调工具，直接返回回答
            return response.content

        for tool_call in response.tool_calls:
            func = tool_map[tool_call.function.name]     # 找到实际函数
            args = json.loads(tool_call.function.arguments)  # 解析参数
            result = func(**args)                         # 执行函数

            # 结果回传 LLM
            messages.append({"role": "tool",
                           "tool_call_id": tool_call.id,
                           "content": json.dumps(result)})
```

关键点：
1. **工具描述要写好**：description 要清晰描述功能和适用场景，让 LLM 能正确判断何时调用
2. **参数校验要严格**：工程侧校验参数类型和范围，防止 LLM 生成非法参数
3. **错误处理要完善**：接口调用失败时将错误信息回传 LLM，让它调整策略

如果是 Java 后端，可以用 Spring Boot 暴露 REST 接口，Agent 通过 HTTP 调用，配合 Redis 缓存和 MySQL 做数据持久化。

---

### Q17：LangChain、LangGraph、CrewAI 这些框架怎么选？

| 框架 | 核心特点 | 适用场景 |
|------|---------|---------|
| **LangChain** | 链式编排、丰富的工具生态 | 快速原型、简单链式任务 |
| **LangGraph** | 图状态机、循环/分支支持 | 复杂工作流、需要循环推理的 Agent |
| **CrewAI** | 角色+目标驱动、开箱即用 | Multi-Agent 快速搭建 |

选型决策：
- **简单链式任务**（如 RAG → 总结 → 翻译）→ LangChain 足够
- **需要循环推理**（如 ReAct 循环、条件分支、回退重试）→ LangGraph
- **快速搭建 Multi-Agent 原型**→ CrewAI
- **生产级复杂 Agent** → LangGraph（对执行流程的控制力最强）

关键认知：框架只是工具，面试考察的是对 Agent 核心概念（ReAct、Memory、Tool Use）的理解深度，而非框架 API 熟练度。

---

### Q18：Agent 执行时出现死循环怎么解决？

死循环是 Agent 工程化中的常见问题，典型表现是 Agent 反复执行相同的 Action 或在几个步骤间来回切换。解决方案分三层：

**预防层**：
- 设置最大迭代次数（max_iterations），超过强制终止
- 设置最大 Token 消耗上限（max_tokens_limit）
- 检测重复 Action：如果连续 N 次调用相同工具且参数相同，触发熔断

**检测层**：
- Action 指纹去重：对 `(tool_name, params_hash)` 做指纹，记录最近 N 次的指纹集合
- 语义相似度检测：用 Embedding 计算当前 Thought 与历史 Thought 的相似度，过高则判定为循环
- 状态机约束：用显式状态机（如 LangGraph）替代自由循环，从结构上避免死循环

**恢复层**：
- 循环检测到后，切换到"总结模式"，让 LLM 总结当前进展并给出最终回答
- 自动重试：清空最近 N 轮历史，从上一个检查点重新开始
- 人工介入：通知人工接管

---

### Q19：AI Coding 工具（如 Cursor、GitHub Copilot）和 Agent 是什么关系？

AI Coding 工具本质上是**垂直领域的 Agent**，但做了特殊优化：

| 维度 | AI Coding 工具 | 通用 Agent |
|------|--------------|-----------|
| 场景 | 代码编写、调试、重构 | 通用任务 |
| 感知 | 代码文件、LSP 信息、Git 状态 | 用户输入、API 返回 |
| 工具 | 文件读写、终端执行、代码搜索 | MCP/Skill 定义的工具集 |
| 记忆 | 项目上下文、代码索引 | Memory 系统 |
| 特殊优化 | AST 感知、增量编辑、语法感知补全 | 通用推理 |

核心认知：Cursor 的 Composer 模式就是一个 Code Agent——感知代码上下文（Read）、规划修改方案（Plan）、执行文件编辑（Act）、根据编译/Lint 结果修正（Observe），这正是 ReAct 循环。理解了 Agent 的核心概念，就能理解 AI Coding 工具的设计原理。

---

### Q20：如何保证 Agent 的可靠性和稳定性？

可靠性是 Agent 落地的核心挑战，需要多层面保障：

**LLM 层**：
- JSON 输出稳定性：使用 Structured Output / JSON Mode 强制格式
- 温度参数控制：需要确定性输出的场景设 temperature=0
- 重试机制：对 LLM API 调用做重试（带指数退避）

**编排层**：
- 状态机替代自由循环：用 LangGraph 等显式状态机约束执行路径
- 检查点机制：每步保存状态，失败可从最近检查点恢复
- 超时控制：每步设置超时，避免卡死

**工具层**：
- 输入校验：调用工具前校验参数完整性和合法性
- 降级策略：工具不可用时切换备选方案
- 结果校验：工具返回结果后做格式和语义校验

**系统层**：
- 全链路 Trace：记录每步的输入、输出、耗时，便于排查
- 熔断器模式：连续失败达到阈值时暂停服务
- 人工兜底：关键决策节点可设置人工确认环节

---

## 四、面试话术

### 话术 1：请介绍一下你对 AI Agent 的理解

> 我理解的 AI Agent 是一种以大语言模型为核心决策引擎的自主智能系统。和传统的 LLM 对话不同，Agent 具备四个关键能力——**感知**用户意图、**规划**执行步骤、**调用工具**完成操作、**反思**执行结果并动态调整。从架构上看，它包含七大核心组件：感知模块、LLM 大脑、规划模块、记忆模块（短期上下文加长期向量库）、执行模块、反馈闭环、以及调度模块。从工程实现角度，核心挑战在于如何稳定地实现 Function Calling 的工具调用循环、如何构建多层次的记忆体系、以及如何做好安全兜底（权限沙箱、超时控制、隐私脱敏）。目前我们团队主要用 LangChain/LangGraph 做编排，工具调用走 MCP 协议标准化，记忆层用 Redis 加 Milvus 向量库的方案。

### 话术 2：Agent 和普通 LLM 有什么区别

> 最核心的区别在于**自主性和闭环能力**。普通 LLM 是"你问我答"——用户给一个问题，模型给一个回答，交互就结束了。而 Agent 是"你说目标，我来搞定"——用户给一个目标，Agent 会自主规划步骤、调用外部工具、观察执行结果、在出错时反思调整，形成一个完整的决策闭环。具体来说，Agent 比 LLM 多了三个东西：第一是**记忆**，能跨轮次保留上下文和经验知识；第二是**工具使用能力**，通过 Function Calling 或 MCP 协议调用外部 API、数据库、搜索引擎等；第三是**规划与反思**，用 ReAct 等范式将推理和行动交织，而不是一次性输出。打个比方，普通 LLM 像一个百科全书——你翻到哪页看哪页，Agent 像一个能自己翻书、查资料、打电话确认的助手。

### 话术 3：Function Calling 是怎么工作的

> Function Calling 的核心思路是让 LLM 只负责"决策"而不负责"执行"。具体流程分五步：第一步，我们在调用 LLM 时传入工具列表（每个工具包含名称、功能描述、参数 Schema）；第二步，LLM 根据用户意图和工具描述做语义匹配，判断是否需要调用工具以及调用哪个；第三步，如果需要调用，LLM 输出一个结构化的 JSON，包含工具名称和参数值；第四步，我们在工程侧解析这个 JSON，实际调用对应的函数或 API；第五步，将执行结果追加回上下文，让 LLM 基于结果生成最终回答。关键点在于 LLM 本身不执行任何操作，它只输出一个"调用意图"的 JSON，真正的执行发生在我们的业务代码中。这样做的好处是安全可控——我们可以在执行前做参数校验、权限检查、审计记录。实际工程中最大的挑战是保证 JSON 输出的稳定性，我们通常用 Schema 约束加 Pydantic 校验加自动重试的三层保障策略。

---

> **相关链接**：
> - [[八股/agent/ReAct]] — Agent 推理范式的核心实现
> - [[八股/agent/MCP]] — 工具调用的标准化协议
> - [[八股/agent/Memory]] — Agent 记忆系统设计
> - [[八股/agent/RAG]] — 检索增强生成技术
> - [[八股/agent/Multi-Agent多智能体系统]] — 多智能体协作
> - [[八股/agent/Agent总览]] — AI Agent 知识体系总览