# 后端要掌握的 AI Agent 知识 — Skill

来源：小红书笔记，作者：程序员流年

---

## 一、核心概念解析：什么是 Skill？

### 1. Skill 的准确定义

Skill 是**结构化的、可复用的能力单元**，将 SOP（标准操作流程）封装起来，指导 Agent 执行。

一个 Skill 本质上是一份**可被 Agent 解析和执行的"操作手册"**，它定义了：
- **什么时候用**（when_to_use）
- **按什么步骤执行**（Steps）
- **需要调用哪些工具**（tools）
- **每个步骤的输入输出和逻辑**（logic）

### 2. Skill 与 Agent 的关系

| 概念 | 角色 | 职责 |
|------|------|------|
| Agent | 调度引擎（Dispatcher） | 意图识别、Skill 路由、上下文管理、结果整合 |
| Skill | 能力库（Capability） | 具体任务的标准化执行流程、工具编排、质量控制 |

可以类比为：Agent 是"项目经理"，Skill 是"标准操作规程（SOP）文档"。

---

## 二、核心价值：Skill 解决了什么问题？

### 1. 稳定性与确定性

LLM 的输出本质上是概率性的，每次生成可能不同。Skill 通过预定义 Steps，为 Agent 提供了确定性的执行框架，将"自由发挥"约束在可控范围内。

### 2. 知识的沉淀与复用

团队的业务知识和最佳实践，往往以文档、代码片段、甚至"口口相传"的形式散落在各处，导致：
- 新人上手慢，知识断层严重
- 同样的坑反复踩，最佳实践难以复用
- 流程变更时难以追踪影响范围

Skill 的解法：Skill 成为了一个承载和复用"领域知识"的绝佳容器。
- **业务流程沉淀**：将成熟业务流程封装为 Skill，新成员无需"口口相传"
- **最佳实践固化**：高效的 Prompt 模式、工具组合固化到 Skill 中，避免重复摸索

### 3. 模块化与可治理性

Skill 将 Agent 的能力体系解耦为一个个独立的、高内聚的模块：
- **独立版本化**：每个 Skill 都可以独立开发、测试、上线和回滚
- **权限与审计**：对每个 Skill 进行细粒度的权限控制
- **跨团队共享**：发布到"技能市场"，其他团队可复用

---

## 三、原理分析：Skill 是如何工作的？

### 1. Skill 的核心构成

```
my_report_skill/
├── SKILL.md        # 必须：定义 Skill 的元信息和执行流程（SOP）
├── scripts/        # 可选：存放 Python/Shell 脚本
│   └── analyze.py
└── assets/         # 可选：存放报告模板、配置文件、示例数据等资源
    └── report_template.docx
```

### 2. SKILL.md 内部结构

#### 元信息（Metadata）
- `name`：Skill 名称
- `description`：功能描述
- `when_to_use`：触发条件（什么时候该用这个 Skill）

#### 操作流程（Steps）
有序指令，每一步是一个原子操作：
- 调用哪个工具
- 传入什么参数
- 输出什么结果
- 下一步做什么

### 3. 与工具（Tool/MCP）和协议的衔接

| 层级 | 职责 | 说明 |
|------|------|------|
| Skill | 流程编排者（What / When） | 决定做什么、什么时候做、按什么顺序做 |
| MCP | 工具的驱动程序（How） | 决定怎么调用工具、数据格式如何转换 |

**Skill 负责业务流程，MCP 负责技术实现。**

### 4. 与 PE、MCP 的分工与边界

架构分层：
- **PE（Prompt Engineering）**：交互层，关注"对话的艺术"
- **MCP**：连接器，关注"能力的接入标准"
- **Skill**：能力核心，关注"如何成事"

| 维度 | PE（提示工程） | MCP（模型上下文协议） | Skill（技能） |
|------|---------------|---------------------|--------------|
| 本质 | 交互技巧 | 通信协议 | 能力封装 |
| 关注点 | 如何与大模型对话 | 如何标准化连接工具 | 如何结构化完成任务 |
| 作用范围 | 单次对话优化 | 工具接入层 | 完整任务流程 |
| 复用性 | 低（每次重新设计） | 中（标准化接口） | 高（可跨场景复用） |
| 粒度 | Prompt 级别 | API 级别 | SOP 级别 |

---

## 四、实践应用：Skill 的落地场景

### 场景一：ServiceAnomaliesAnalysis Skill（SRE 故障根因分析）

触发条件：当用户报告服务异常或告警触发时自动激活。

执行步骤：
1. **收集告警信息**：从 Prometheus/Grafana 拉取异常时间段指标
2. **查询日志**：从 ELK/Loki 中检索异常时间段相关日志
3. **检查变更记录**：查询最近的部署/配置变更
4. **分析依赖链路**：从链路追踪系统中获取上下游调用状态
5. **交叉比对**：将日志、指标、变更记录进行交叉分析
6. **生成结论**：输出可能的根因、影响范围和建议的修复方案
7. **通知相关方**：将分析结果发送给对应的 oncall 人员

工具依赖：Prometheus API、ELK API、CI/CD API、Trace API、通知 API

### 场景二：UserFeedbackInsight Skill（用户负反馈洞察）

触发条件：当需要分析用户反馈、评价、投诉等内容时激活。

执行步骤：
1. **采集反馈数据**：从客服系统、评价系统拉取指定时间段数据
2. **分类与标注**：对反馈进行情感分析和主题分类
3. **趋势分析**：统计各类问题的频率变化趋势
4. **根因归纳**：基于分类结果提炼核心问题
5. **生成洞察报告**：输出问题清单、优先级排序和改进建议

### 场景三：面试安排协调 Skill

触发条件：当用户需要安排面试、协调面试官和候选人时间时激活。

执行步骤：
1. **解析需求**：提取候选人信息、面试岗位、面试轮次
2. **查询面试官日程**：从日历系统中查询可用时间段
3. **时间匹配**：找到候选人和面试官的共同可用时段
4. **发送邀请**：创建日历事件并发送邮件通知
5. **确认与提醒**：面试前一天自动发送提醒

### 复杂逻辑在 Skill 中的处理

**核心原则**：保持 Skill 步骤的"声明式"，将复杂的"命令式"控制流下沉到外部脚本中。

**实现方式一：为 Skill DSL 扩展最小控制流子集**
- YAML if/else 示例（check_user_status → conditional_logic）
- 适合简单二元分支，不适合循环

**实现方式二：调用外部脚本（推荐）**
- 关注点分离、可测试性、表达能力强、可观测性

Skill 步骤调用外部脚本示例：
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

外部脚本通过 stdin/stdout 与 Skill 交互：
```python
# scripts/process_data.py
import sys
import json

def process(data):
    # 复杂业务逻辑处理
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

---

## 五、问题与解答

### Q1：PE、Rule、Skill 和 MCP 如何协同工作？

协同流程：

```
用户输入 → 意图识别 → 前置检查(Rule) → 任务执行(Skill+MCP) → 后处理(Rule) → 输出
```

详细步骤：
1. **意图识别**：Agent 识别用户意图，决定调用哪个 Skill
2. **前置检查（Rule）**：安全规则、合规检查、权限验证
3. **任务执行（Skill + MCP）**：Skill 编排执行流程，MCP 调用具体工具
4. **后处理（Rule）**：输出格式化、敏感信息过滤、审计日志记录

### Q2：Skill 执行失败时的失败处理和兜底机制？

4 种策略：
1. **步骤级重试**：对失败的步骤自动重试（带指数退避）
2. **显式错误分支**：在 Skill 中定义 `on_error` 分支，指定失败后的备选路径
3. **优雅降级**：当某个工具不可用时，切换到备选方案（如用缓存数据替代实时查询）
4. **全局兜底与日志**：所有失败的执行都会记录详细日志，便于排查和优化

### Q3：什么样的任务适合/不适合沉淀为 Skill？

| 适合沉淀为 Skill | 不适合沉淀为 Skill |
|------------------|-------------------|
| 有明确 SOP 的重复性任务 | 一次性的、无规律的请求 |
| 涉及多步骤、多工具编排的任务 | 简单的单步骤问答 |
| 团队高频使用的业务流程 | 纯创意性的、需要自由发挥的任务 |
| 需要质量控制和可追溯的任务 | 高度不确定、流程不稳定的探索性任务 |

### Q4：when_to_use 如何设计？

多信号触发策略：

| 信号类型 | 说明 | 示例 |
|----------|------|------|
| 正例关键词 | 用户输入中包含特定关键词 | "服务挂了"、"5xx 飙升" |
| 领域特征 | 输入属于特定领域 | SRE/运维/监控相关 |
| 上下文线索 | 当前对话上下文暗示 | 用户刚查了某服务的指标 |
| 用户画像 | 用户角色和权限 | SRE 工程师更可能触发故障分析 Skill |
| 结构化输入 | 输入匹配特定格式 | JSON 格式的告警 payload |
| 负触发清单 | 明确排除的场景 | "帮我写代码" 不应触发故障分析 |

### Q5：当两个 Skill 的 when_to_use 发生重叠时，Agent 应如何处理？

路由决策策略：

1. **静态优先级矩阵（Static Priority Matrix）**：为每个 Skill 设置优先级值（1-100）
2. **置信度打分与阈值（Confidence Scoring & Threshold）**：分配置信度分数（0-1），超过 0.9 直接选择
3. **轮次打破规则（Tie-breaking）**：最近使用、角色偏好、上下文相关性、多轮对话历史
4. **组合执行与回退（Composition & Fallback）**：可同时执行多个 Skill 或回退到优先级较低的

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

### Q6：在 Skill 的一个步骤中进行复杂逻辑判断？

**核心原则**：保持 Skill 步骤的"声明式"，将复杂的"命令式"控制流下沉到外部脚本中。

**实现方式一：为 Skill DSL 扩展最小控制流子集**
- YAML if/else 示例（check_user_status → conditional_logic）
- 适合简单二元分支，不适合循环

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

**实现方式二：调用外部脚本（推荐）**

优势：
- **关注点分离**：Skill 定义流程，脚本处理逻辑
- **可测试性**：脚本可以独立测试
- **表达能力强**：Python 可以处理任意复杂逻辑
- **可观测性**：脚本的输入输出可以独立记录和调试

---

## 总结

Skill 是 AI Agent 能力体系的核心，它将"如何完成一个任务"从模糊的 Prompt 指令转变为结构化的 SOP。核心要点：

1. **定义**：Skill 是结构化的、可复用的能力单元，封装 SOP 指导 Agent 执行
2. **价值**：提供确定性、沉淀知识、模块化治理
3. **架构**：SKILL.md（元信息 + Steps）+ 可选的 scripts/ 和 assets/
4. **与 MCP 的关系**：Skill 编排"做什么"，MCP 实现"怎么做"
5. **落地场景**：SRE 故障分析、用户反馈洞察、面试安排等结构化任务
6. **设计要点**：when_to_use 多信号触发、冲突路由策略、失败兜底机制、复杂逻辑下沉到外部脚本
