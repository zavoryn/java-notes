# AI 排障系统设计

## 核心思路

用一个专用 Agent + 持久化记忆库，把每一次 debug 的过程和结论沉淀下来。下次遇到类似报错，先查记忆库，命中就能秒排。

## 架构

```
用户报 bug / 贴报错信息
        │
        ▼
┌─────────────────┐
│  AI 排障 Agent   │  ← 注入专用 skill，带排障 SOP
│  (Claude Code)   │
└───────┬─────────┘
        │
        ├── 1. 查记忆库（向量检索 / 关键词匹配）
        │     命中 → 直接返回历史排障方案 + 验证是否适用
        │     未命中 → 进入完整排障流程
        │
        ├── 2. 完整排障流程
        │     a. 分析报错信息，提取关键特征（异常类型、堆栈、环境）
        │     b. 定位根因（日志分析、代码追踪、复现路径）
        │     c. 给出修复方案
        │     d. 验证修复
        │
        └── 3. 排障完成 → 写入记忆库
              结构化记录：症状、根因、方案、耗时
```

## 记忆库设计

### 方案 A：MD 文件仓库（轻量，零依赖）

每个 bug 一个 MD 文件，按分类建目录：

```
troubleshooting/
├── java/
│   ├── NullPointer_分页查询未判空.md
│   ├── OOM_大文件导出.md
│   └── ClassNotFound_依赖冲突.md
├── spring/
│   ├── Bean创建失败_循环依赖.md
│   └── 事务不生效_自调用.md
├── mysql/
│   ├── 死锁_并发更新.md
│   └── 慢查询_缺失索引.md
├── devops/
│   ├── Docker网络不通.md
│   └── K8s_Pending_资源不足.md
└── _index.md          ← 总索引，用于快速关键词匹配
```

**单条记录模板：**

```markdown
---
id: BUG-2026-001
date: 2026-05-16
tags: [java, NullPointerException, MyBatis-Plus, 分页]
severity: medium      # low / medium / high / critical
resolved: true
---

## 症状
一句话描述表面现象。

## 报错信息
\```
关键堆栈 / 日志片段
\```

## 根因
一句话描述真正的原因。

## 排查路径
1. 第一步做了什么，发现了什么
2. 第二步做了什么，发现了什么
3. ...

## 修复方案
具体的修复代码或配置变更。

## 耗时
从开始排查到解决用了多久。

## 教训
一句话总结，下次怎么避免。
```

### 方案 B：向量数据库（精准语义检索）

| 选项 | 适用场景 | 优劣 |
|------|---------|------|
| ChromaDB | 本地开发，Python 生态 | 轻量，零配置，但功能单一 |
| Milvus / Qdrant | 生产级，需要高并发 | 功能全，但部署重 |
| SQLite + fts5 | 超轻量，纯全文检索 | 无语义匹配，但够用 |
| 飞书多维表格 | 不想搭服务，用现成工具 | 读写方便，但检索能力有限 |

**推荐起步方案：MD 仓库 + SQLite fts5 全文索引**
- MD 文件人类可读，git 可追踪
- SQLite 做一层索引，支持快速模糊搜索
- 等积累到 500+ 条再考虑上向量数据库

## 排障 Agent Skill 设计

### Skill 定义（伪代码）

```yaml
name: troubleshoot
description: AI 排障助手，查记忆库 + 排 bug + 沉淀经验
trigger: 用户报告 bug / 贴报错 / 说"排查一下"

steps:
  - name: 提取特征
    action: 从用户输入中提取异常类型、技术栈、关键词

  - name: 检索记忆库
    action: |
      方案A: grep 关键词在 troubleshooting/ 下搜索
      方案B: 向量检索 top-5 相似案例
    output: 匹配的历史案例列表

  - name: 判断是否命中
    condition: 匹配度 > 阈值
    on_hit:
      - 展示历史案例
      - 确认是否适用当前情况
      - 如果适用，直接用历史方案
    on_miss:
      - 进入完整排障 SOP

  - name: 排障 SOP
    steps:
      - 复现问题（看日志、看代码）
      - 缩小范围（二分法、对比法）
      - 定位根因
      - 提出修复方案
      - 验证修复

  - name: 沉淀记录
    action: 按模板生成 MD 文件，写入记忆库
    fields: [症状, 报错信息, 根因, 排查路径, 修复方案, 教训]
```

### Agent 行为注入

通过 CLAUDE.md 或 skill 配置，给排障 Agent 注入以下行为：

1. **每次排障开始前**：先查记忆库，展示匹配结果
2. **每次排障结束后**：主动询问是否沉淀到记忆库
3. **定期回顾**：每周/每月汇总高频 bug，提炼模式

## 实施步骤

### Phase 1：最小可用（1 天）

- [ ] 创建 `troubleshooting/` 目录和模板
- [ ] 在项目 CLAUDE.md 中加入排障指令
- [ ] 手动录入 3-5 条历史 bug 作为种子数据
- [ ] 用 grep 做最基础的关键词检索

### Phase 2：半自动化（1 周）

- [ ] 写一个排障 skill，自动执行查记忆库 + 写入记录
- [ ] 搭建 SQLite fts5 索引，支持模糊搜索
- [ ] 加自动去重（相似度检测，避免重复记录）

### Phase 3：智能化（按需）

- [ ] 接入向量数据库，语义检索
- [ ] 排障 Agent 自动关联相似案例
- [ ] 定期生成"高频 bug Top 10"报告
- [ ] 把排障经验反哺到代码规范 / CI 检查中

## 现在就能用的简易版

不用搭任何服务，直接利用 Claude Code 现有能力：

1. **记忆库**：用本项目 memory 系统（`C:\Users\ming\.claude\projects\...\memory\`），每条排障经验写一个 MD
2. **检索**：Claude Code 每次对话自动加载 MEMORY.md，排障时先读 memory
3. **写入**：排障结束后，用 feedback 类型 memory 存储排障经验

```markdown
<!-- memory 中的排障记录示例 -->
---
name: bug-mybatis-plus-pagination-npe
description: MyBatis-Plus 分页查询返回空列表时直接 get 触发 NPE
type: feedback
---

MyBatis-Plus 的 Page.getRecords() 在无数据时返回空列表而非 null，
但如果用自定义 mapper 返回 Page 对象且未正确配置，
可能返回 null。排查时先确认 mapper 返回类型。

**Why:** 线上出现过一次，排查了 2 小时才发现是返回值未判空。
**How to apply:** 凡是 MyBatis-Plus 分页查询，先判 records 是否为 null。
```

这是零成本方案，立刻就能开始用。
