---
tags:
  - 八股
  - AI
  - Agent
status: 待复习
created: 2026-05-16
---

# Skill 技能系统

> Skill 是 AI Agent 能力体系的核心，将"如何完成一个任务"从模糊的 Prompt 指令转变为结构化的 SOP。本文从定义、核心价值、内部结构、落地场景到设计要点全面讲解。

---

## 一、知识点讲解

### 1. Skill 定义

Skill 是**结构化的、可复用的能力单元**，将 SOP（标准操作流程）封装起来，指导 Agent 执行。

一个 Skill 本质上是一份**可被 Agent 解析和执行的"操作手册"**，它定义了：

| 要素 | 说明 | 类比 |
|------|------|------|
| when_to_use | 什么时候用 | SOP 的触发条件 |
| Steps | 按什么步骤执行 | SOP 的操作流程 |
| tools | 需要调用哪些工具 | SOP 的工具清单 |
| logic | 每步的输入输出和逻辑 | SOP 的操作规范 |

### 2. Skill 与 Agent 的关系

| 概念 | 角色 | 职责 |
|------|------|------|
| **Agent** | 项目经理（Dispatcher） | 意图识别、Skill 路由、上下文管理、结果整合 |
| **Skill** | SOP 文档（Capability） | 具体任务的标准化执行流程、工具编排、质量控制 |

```
┌──────────────────────────────────────────────────────────┐
│                        Agent（项目经理）                    │
│                                                          │
│   用户输入 → 意图识别 → 路由到 Skill → 整合结果 → 返回    │
│                    │                                     │
│         ┌──────────┼──────────┐                          │
│         ▼          ▼          ▼                          │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐                │
│   │ Skill A  │ │ Skill B  │ │ Skill C  │                │
│   │ (故障分析)│ │ (反馈洞察)│ │ (面试安排)│                │
│   │          │ │          │ │          │                │
│   │ Step 1   │ │ Step 1   │ │ Step 1   │                │
│   │ Step 2   │ │ Step 2   │ │ Step 2   │                │
│   │ Step 3   │ │ Step 3   │ │ Step 3   │                │
│   │ ...      │ │ ...      │ │ ...      │                │
│   └──────────┘ └──────────┘ └──────────┘                │
└──────────────────────────────────────────────────────────┘
```

### 3. Skill 核心价值

#### 3.1 稳定性与确定性

LLM 的输出本质上是概率性的，每次生成可能不同。Skill 通过预定义 Steps，为 Agent 提供了**确定性的执行框架**，将"自由发挥"约束在可控范围内。

#### 3.2 知识沉淀与复用

| 问题 | Skill 的解法 |
|------|-------------|
| 新人上手慢，知识断层严重 | 将成熟业务流程封装为 Skill，无需"口口相传" |
| 同样的坑反复踩 | 高效的 Prompt 模式、工具组合固化到 Skill 中 |
| 流程变更难以追踪 | Skill 版本化，变更影响范围可控 |

#### 3.3 模块化与可治理性

- **独立版本化**：每个 Skill 可独立开发、测试、上线和回滚
- **权限与审计**：对每个 Skill 进行细粒度的权限控制
- **跨团队共享**：发布到"技能市场"，其他团队可复用

### 4. SKILL.md 内部结构

```
my_report_skill/
├── SKILL.md        # 必须：定义 Skill 的元信息和执行流程（SOP）
├── scripts/        # 可选：存放 Python/Shell 脚本
│   └── analyze.py
└── assets/         # 可选：存放报告模板、配置文件、示例数据等资源
    └── report_template.docx
```

#### SKILL.md 核心组成

```yaml
# ============ 元信息（Metadata） ============
name: "ServiceAnomaliesAnalysis"
description: "SRE 故障根因分析 Skill"
when_to_use:
  - 正例关键词: ["服务挂了", "5xx飙升", "响应超时", "告警触发"]
  - 领域特征: ["SRE", "运维", "监控", "故障"]
  - 上下文线索: ["用户刚查了某服务的指标"]
  - 用户画像: ["SRE工程师", "运维人员"]

# ============ 操作流程（Steps） ============
steps:
  - name: collect_metrics
    tool: prometheus.query
    params:
      time_range: "last_1h"
    output: metrics_data

  - name: query_logs
    tool: elasticsearch.search
    params:
      index: "app-logs-*"
      query: "{{metrics_data.anomaly_time_range}}"
    output: log_results

  - name: check_deployments
    tool: cicd.api.query_changes
    params:
      time_range: "last_24h"
    output: change_records

  - name: analyze_root_cause
    tool: llm.analyze
    params:
      input:
        metrics: "{{metrics_data}}"
        logs: "{{log_results}}"
        changes: "{{change_records}}"
    output: root_cause_analysis

  - name: notify_oncall
    tool: notification.send
    params:
      channel: "#sre-alert"
      message: "{{root_cause_analysis}}"
```

### 5. Skill 与 PE、MCP 的分工边界

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  PE（Prompt Engineering）   MCP              Skill         │
│       交互层              连接器            能力核心         │
│   "对话的艺术"         "能力的接入标准"    "如何成事"        │
│                                                            │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────┐    │
│  │ 关注怎么和   │   │ 关注怎么标准  │   │ 关注怎么结构  │    │
│  │ 大模型对话   │   │ 化连接工具    │   │ 化完成任务    │    │
│  └─────────────┘   └──────────────┘   └──────────────┘    │
│       ↓                  ↓                  ↓              │
│   单次对话优化       工具接入层          完整任务流程        │
│   复用性：低          复用性：中          复用性：高         │
│   粒度：Prompt级     粒度：API级         粒度：SOP级       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

| 维度 | PE（提示工程） | MCP（模型上下文协议） | Skill（技能） |
|------|---------------|---------------------|--------------|
| 本质 | 交互技巧 | 通信协议 | 能力封装 |
| 关注点 | 如何与大模型对话 | 如何标准化连接工具 | 如何结构化完成任务 |
| 作用范围 | 单次对话优化 | 工具接入层 | 完整任务流程 |
| 复用性 | 低（每次重新设计） | 中（标准化接口） | 高（可跨场景复用） |
| 粒度 | Prompt 级别 | API 级别 | SOP 级别 |

### 6. Skill 落地场景

#### 场景一：SRE 故障根因分析（ServiceAnomaliesAnalysis）

| 要素 | 内容 |
|------|------|
| 触发条件 | 用户报告服务异常或告警触发 |
| 执行步骤 | 收集告警 → 查询日志 → 检查变更 → 分析依赖链路 → 交叉比对 → 生成结论 → 通知相关方 |
| 工具依赖 | Prometheus API、ELK API、CI/CD API、Trace API、通知 API |

#### 场景二：用户反馈洞察（UserFeedbackInsight）

| 要素 | 内容 |
|------|------|
| 触发条件 | 需要分析用户反馈、评价、投诉等内容 |
| 执行步骤 | 采集数据 → 分类标注 → 趋势分析 → 根因归纳 → 生成洞察报告 |

#### 场景三：面试安排协调

| 要素 | 内容 |
|------|------|
| 触发条件 | 需要安排面试、协调面试官和候选人时间 |
| 执行步骤 | 解析需求 → 查询日程 → 时间匹配 → 发送邀请 → 确认提醒 |

### 7. 复杂逻辑处理：声明式 Skill + 命令式脚本

**核心原则**：保持 Skill 步骤的"声明式"，将复杂的"命令式"控制流下沉到外部脚本中。

#### 方式一：YAML if/else（适合简单二元分支）

```yaml
steps:
  - name: check_user_status
    tool: user.get_status
    output: user_status

  - name: conditional_logic
    if: "{{user_status}} == 'vip'"
    then:
      - name: apply_vip_discount
        tool: discount.apply
        params:
          rate: 0.8
    else:
      - name: apply_normal_price
        tool: price.calculate
```

#### 方式二：调用外部脚本（推荐，适合复杂逻辑）

```yaml
steps:
  - name: get_raw_data
    tool: data.api.fetch
    output: raw_data

  - name: process_data_with_script
    tool: script.run
    params:
      path: "./scripts/process_data.py"
      input_data: "{{raw_data}}"
    output: processed_results

  - name: render_report
    tool: report.generate
    params:
      data: "{{processed_results}}"
```

外部脚本示例：

```python
# scripts/process_data.py
import sys, json

def process(data):
    result = []
    for item in data:
        if item["score"] > 0.8:
            result.append({"id": item["id"], "label": "high_risk"})
    return result

if __name__ == "__main__":
    input_data = json.loads(sys.stdin.read())
    output = process(input_data)
    print(json.dumps(output))
```

| 对比 | YAML if/else | 外部脚本 |
|------|-------------|---------|
| 适用场景 | 简单二元分支 | 复杂循环、条件嵌套 |
| 关注点分离 | 弱 | 强（Skill 定义流程，脚本处理逻辑） |
| 可测试性 | 低 | 高（脚本可独立测试） |
| 表达能力 | 有限 | 完整（Python 可处理任意逻辑） |

### 8. Skill 失败处理

| 策略 | 说明 | 类比后端 |
|------|------|---------|
| **步骤级重试** | 对失败步骤自动重试（带指数退避） | 类似 Spring Retry `@Retryable` |
| **显式错误分支** | 在 Skill 中定义 `on_error` 分支 | 类似异常处理的 `catch` 块 |
| **优雅降级** | 工具不可用时切换备选方案（如缓存替代实时查询） | 类似 Hystrix/Sentinel 降级 |
| **全局兜底与日志** | 所有失败记录详细日志 | 类似全局异常处理器 + ELK 日志 |

### 9. when_to_use 多信号触发设计

| 信号类型 | 说明 | 示例 |
|----------|------|------|
| **正例关键词** | 用户输入包含特定关键词 | "服务挂了"、"5xx 飙升" |
| **领域特征** | 输入属于特定领域 | SRE/运维/监控相关 |
| **上下文线索** | 当前对话上下文暗示 | 用户刚查了某服务的指标 |
| **用户画像** | 用户角色和权限 | SRE 工程师更可能触发故障分析 |
| **结构化输入** | 输入匹配特定格式 | JSON 格式的告警 payload |
| **负触发清单** | 明确排除的场景 | "帮我写代码"不应触发故障分析 |

### 10. Skill 冲突路由策略

当两个 Skill 的 when_to_use 发生重叠时：

```
用户意图
    │
    ▼
┌──────────────────┐
│ 匹配所有 when_to_use│
│ 命中的 Skill 列表   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     置信度 > 0.9
│ ① 静态优先级排序   │────────────────→ 直接执行最高优先级 Skill
│ （priority 1-100）│
└────────┬─────────┘
         │ 置信度差距 < 0.1
         ▼
┌──────────────────┐
│ ② 置信度打分       │
│ （0-1 阈值判定）   │
└────────┬─────────┘
         │ 仍然无法区分
         ▼
┌──────────────────┐
│ ③ 轮次打破规则     │
│ 最近使用/角色偏好/  │
│ 上下文相关性       │
└────────┬─────────┘
         │ 仍然无法区分
         ▼
┌──────────────────┐
│ ④ 组合执行或回退   │
│ 同时执行/询问用户   │
└──────────────────┘
```

路由决策伪代码：

```python
def route_skill(user_intent, matched_skills):
    if not matched_skills:
        return "Sorry, I can't help with that."

    sorted_skills = sorted(
        matched_skills,
        key=lambda x: (x.priority, x.confidence_score),
        reverse=True
    )
    top_skill = sorted_skills[0]

    if top_skill.confidence_score > 0.9:
        return f"Executing {top_skill.name}"

    if len(sorted_skills) > 1:
        second_skill = sorted_skills[1]
        if (top_skill.confidence_score - second_skill.confidence_score) < 0.1:
            return f"Which one do you mean? 1. {top_skill.name} or 2. {second_skill.name}"

    return f"I'm guessing you mean {top_skill.name}. Let's proceed. (If not, say 'stop')"
```

---

## 二、串联理解记忆

### Skill 与 MCP

| 维度 | Skill | MCP |
|------|-------|-----|
| 角色 | 能力的"操作手册" | 工具的"驱动程序" |
| 回答的问题 | **做什么、什么时候做**（What & When） | **怎么做**（How） |
| 类比 | 业务 Service 层的 SOP | JDBC 驱动 / API Client |

> **协同关系**：Skill 的 Steps 中定义了"需要调用哪个工具"，底层的工具调用走 MCP 协议。Skill 编排"做什么"，MCP 实现"怎么做"。

### Skill 与 ReAct

在 [[八股/agent/2-ReAct推理范式]] 中，ReAct 的 Action 步骤是"原子化"的。**Skill 的 Steps 定义了 ReAct 中 Action 的具体执行流程**——一个 Skill 就是一组预编排的 Action 序列。

```
ReAct 循环:  Thought → Action → Observation → Thought → Action → ...
Skill Steps:  Step1(Tool A) → Step2(Tool B) → Step3(LLM分析) → ...
                     ↑                 ↑
              ReAct 中的 Action    ReAct 中的 Action
```

### Skill 与 Memory

在 [[八股/agent/6-Memory记忆系统]] 中，程序记忆（L4）存储"工具使用方法、SOP"。**Skill 执行过程中的经验沉淀为程序记忆**——例如某个 Skill 步骤经常失败，系统可以记录这个"教训"，下次优化 Skill 流程。

```
Skill 执行 → 经验沉淀 → 程序记忆（L4）
    ↓                         ↓
执行历史                 "Step3 调用 API X 经常超时，
                         建议增加重试逻辑"
```

---

## 三、面试题

### Q1：什么是 Skill？和 Agent 的关系是什么？

Skill 是结构化的、可复用的能力单元，将 SOP 封装起来指导 Agent 执行。Agent 是"项目经理"，负责意图识别和 Skill 路由；Skill 是"SOP 文档"，负责具体任务的标准化执行流程。一个 Skill 定义了 when_to_use（什么时候用）、Steps（按什么步骤执行）、tools（需要调用哪些工具）。

### Q2：PE、Rule、Skill 和 MCP 如何协同工作？

协同流程：`用户输入 → 意图识别 → 前置检查(Rule) → 任务执行(Skill+MCP) → 后处理(Rule) → 输出`

- **PE**（交互层）：关注"对话的艺术"，优化单次交互质量
- **Rule**（守卫层）：安全规则、合规检查、权限验证
- **Skill**（能力核心）：编排"做什么"，定义完整的 SOP
- **MCP**（连接器）：实现"怎么做"，标准化工具调用

### Q3：Skill 执行失败时的兜底机制？

4 种策略层层递进：
1. **步骤级重试**：对失败步骤自动重试（带指数退避）
2. **显式错误分支**：在 Skill 中定义 `on_error` 分支，指定失败后备选路径
3. **优雅降级**：工具不可用时切换备选方案（如缓存数据替代实时查询）
4. **全局兜底与日志**：所有失败记录详细日志，便于排查优化

### Q4：什么样的任务适合/不适合沉淀为 Skill？

| 适合沉淀为 Skill | 不适合沉淀为 Skill |
|------------------|-------------------|
| 有明确 SOP 的重复性任务 | 一次性的、无规律的请求 |
| 涉及多步骤、多工具编排的任务 | 简单的单步骤问答 |
| 团队高频使用的业务流程 | 纯创意性的、需要自由发挥的任务 |
| 需要质量控制和可追溯的任务 | 高度不确定、流程不稳定的探索性任务 |

### Q5：when_to_use 如何设计？

采用多信号触发策略：
- **正例关键词**：用户输入中包含特定关键词（如"服务挂了"）
- **领域特征**：输入属于特定领域（如 SRE/运维）
- **上下文线索**：当前对话上下文暗示（如用户刚查了指标）
- **用户画像**：用户角色和权限（如 SRE 工程师）
- **结构化输入**：输入匹配特定格式（如 JSON 告警 payload）
- **负触发清单**：明确排除的场景

### Q6：两个 Skill 的 when_to_use 重叠时，Agent 如何处理？

路由决策四步走：
1. **静态优先级矩阵**：为每个 Skill 设置优先级值（1-100），高优先级优先
2. **置信度打分与阈值**：分配置信度分数（0-1），超过 0.9 直接选择
3. **轮次打破规则**：综合考虑最近使用、角色偏好、上下文相关性
4. **组合执行与回退**：无法区分时询问用户或同时执行

### Q7：什么是渐进式披露（Progressive Disclosure）？为什么它在 Skill 设计中很重要？

渐进式披露是一种信息管理策略——**不一次性把所有信息给 LLM，而是根据任务进展按需提供**。

```
传统方式（信息轰炸）:
  Prompt = 完整工具列表(50个) + 全部文档 + 完整历史
  → LLM 在海量信息中迷失，选择困难，Token 浪费严重

渐进式披露（按需加载）:
  Step 1: 只给任务概述 + 3个最相关的工具
  Step 2: 工具调用后，按需加载相关文档片段
  Step 3: 根据中间结果，动态决定是否需要更多信息
```

在 Skill 设计中的体现：
- **工具描述分层**：Skill 的 when_to_use 只给简要描述，匹配后再展开详细步骤
- **上下文按需加载**：Skill Steps 中只在需要时才检索 RAG 或查询历史
- **条件分支**：根据中间结果决定后续步骤，而非一开始就加载所有可能路径

好处：① 节省 Token 消耗；② 减少 Lost in the Middle 问题；③ 提高工具选择准确率。

### Q8：为什么用 Skill 封装而不是直接把所有工具暴露给 LLM？

直接暴露所有工具的问题：

| 问题 | 说明 |
|------|------|
| **选择困难** | 50+ 个工具描述塞入 Prompt，LLM 选错概率高 |
| **Token 浪费** | 每次都带全部工具描述，大量 Token 浪费在无关工具上 |
| **缺乏编排** | LLM 自由组合工具可能走弯路，Skill 预定义最优路径 |
| **质量失控** | 没有标准流程，每次执行路径不同，结果不一致 |
| **知识断层** | 新人不知道怎么组合工具，知识只存在老员工脑子里 |

Skill 的解决方式：
```
无 Skill:  LLM → 从50个工具中选 → 可能选错 → 多次试错 → 结果不稳定
有 Skill:  LLM → 路由到 Skill → Skill 提供3个最相关工具 → 按预定义SOP执行 → 结果稳定
```

类比：Skill 就像是"套餐"——餐厅不会让客人自己从100种食材中自由搭配（选择困难+质量失控），而是设计好几种套餐（Skill），客人只需要说"我要套餐A"。

### Q9：Skill 和 Rule 有什么区别？它们如何协同？

| 维度 | Skill | Rule |
|------|-------|------|
| 本质 | 能力单元（SOP） | 约束条件（守卫） |
| 回答的问题 | **做什么、怎么做** | **不能做什么、必须检查什么** |
| 执行时机 | 意图匹配后执行 | Skill 执行前后拦截 |
| 类比 | 业务 Service 层 | 拦截器/过滤器 |

协同流程：
```
用户输入 → 意图识别 → 前置 Rule 检查 → Skill 执行 → 后置 Rule 检查 → 输出
                         ↑                        ↑
                    安全规则                   内容审核
                    权限验证                   格式校验
                    输入过滤                   敏感信息过滤
```

典型 Rule 示例：
- **安全 Rule**：禁止调用删除数据库的工具
- **合规 Rule**：输出必须经过敏感信息过滤
- **权限 Rule**：普通用户不能访问管理员级别的 Skill
- **格式 Rule**：返回 JSON 必须符合指定 Schema

---

## 四、面试话术

### 话术 1："请介绍一下你对 Agent Skill 系统的理解"

> Skill 是 AI Agent 能力体系的核心概念。它是一个结构化的、可复用的能力单元，本质上是一份可被 Agent 解析和执行的 SOP 文档。
>
> Agent 和 Skill 的关系可以用一个类比来理解：Agent 是项目经理，负责意图识别和任务路由；Skill 是标准操作规程文档，定义了具体任务的执行步骤。一个 Skill 定义了四个要素：when_to_use（触发条件）、Steps（执行流程）、tools（工具依赖）和 logic（输入输出逻辑）。
>
> Skill 的核心价值有三点：第一是稳定性，通过预定义 Steps 为 Agent 提供确定性执行框架，约束 LLM 的"自由发挥"；第二是知识沉淀，将团队最佳实践封装为可复用的 Skill，避免知识散落和重复踩坑；第三是模块化治理，每个 Skill 可独立版本化、权限控制和跨团队共享。
>
> 在架构层面，Skill 与 MCP 和 PE 形成三层分工：PE 是交互层，关注对话质量；MCP 是连接器，关注工具接入标准化；Skill 是能力核心，关注如何结构化完成任务。Skill 编排"做什么"，MCP 实现"怎么做"。
>
> 在设计层面，有两个关键问题：when_to_use 的多信号触发设计（正例关键词 + 领域特征 + 上下文线索 + 用户画像），以及 Skill 冲突时的路由策略（静态优先级 → 置信度打分 → 轮次打破 → 组合执行/询问用户）。对于复杂逻辑，采用声明式 Skill + 命令式外部脚本的组合，保持 Skill 的可读性和可测试性。

---

**相关笔记**：
- [[八股/agent/4-MCP协议]] — Skill 编排"做什么"，MCP 实现"怎么做"
- [[八股/agent/2-ReAct推理范式]] — Skill 的 Steps 定义了 ReAct 中 Action 的具体执行流程
- [[八股/agent/6-Memory记忆系统]] — Skill 执行过程中的经验沉淀为程序记忆（L4）
