---
tags:
  - 八股
  - 消息队列
  - Kafka
  - Kafka消费者
status: 学习中
created: 2026-05-06
---

# Kafka 消费者机制

> Consumer 的核心在于**消费者组的分区分配**和 **Offset 提交策略**——这是最容易出线上事故的地方。

---

## 一、消费者组（Consumer Group）

### 1.1 核心规则

```
Topic: order-topic (6 Partitions)
  Consumer Group: order-group (3 Consumers)
    Consumer-1 → P0, P1
    Consumer-2 → P2, P3
    Consumer-3 → P4, P5

规则:
  1. 同一 Group 内，一个 Partition 只能被一个 Consumer 消费
  2. 一个 Consumer 可以消费多个 Partition
  3. Consumer 数 > Partition 数 → 多出的 Consumer 空闲
  4. 不同 Group 之间完全独立，各自维护 Offset
```

### 1.2 为什么需要消费者组？

| 场景 | 方式 |
|------|------|
| **广播** | 不同 Consumer Group 订阅同一 Topic → 各自消费全量 |
| **负载均衡** | 同一 Group 内多个 Consumer 分摊 Partition |
| **容错** | 某 Consumer 挂了 → Partition 自动分配给其他 Consumer |

---

## 二、Rebalance（重平衡）—— 核心机制

> 消费者组内分区分配发生变化的重新分配过程。

### 2.1 触发条件

| 条件 | 说明 |
|------|------|
| Consumer 加入/离开 | 新实例启动或旧实例挂掉 |
| Topic 分区变化 | 动态增加了 Partition |
| 消费超时 | `max.poll.interval.ms` 内未完成 poll → 被踢出 |

### 2.2 Rebalance 的危害

```
正常消费: ████████████████████████
发生 Rebalance:
  组协调 → 停止消费 → 重新分配分区 → 恢复消费
  ████████░░░░░░░░░░░░███████████████
               ↑ Stop-The-World 停顿！

频繁 Rebalance → 消费断断续续 → 积压越来越严重
```

### 2.3 Rebalance 排查与解决

| 原因 | 排查方法 | 解决方案 |
|------|---------|---------|
| **处理太慢** | 检查 `max.poll.interval.ms` 是否被超 | 增大参数或优化业务逻辑 |
| **GC 停顿** | 检查 JVM GC 日志 | 调优 GC 参数 |
| **网络闪断** | `session.timeout.ms` 太小 | 适当增大（如 30s→60s） |
| **心跳丢失** | `heartbeat.interval.ms` 不合理 | 设为 `session.timeout.ms` 的 1/3 |

```java
// 关键参数
props.put("max.poll.interval.ms", 300000);   // 两次 poll 最大间隔（太久处理不完会被踢）
props.put("session.timeout.ms", 45000);       // 心跳超时（Broker 认为 Consumer 挂了）
props.put("heartbeat.interval.ms", 15000);    // 心跳间隔（session.timeout 的 1/3）
```

---

## 三、Offset 提交策略

> Offset 提交方式直接影响**消息可靠性和重复消费**，是最容易踩坑的地方。

### 3.1 自动提交 vs 手动提交

```java
// 自动提交（开箱即用，但有坑）
props.put("enable.auto.commit", true);
props.put("auto.commit.interval.ms", 5000);  // 每 5 秒自动提交

// 手动提交（推荐，精确控制）
props.put("enable.auto.commit", false);
```

### 3.2 自动提交的两个经典问题

```
自动提交导致消息丢失:
  1. poll() 拉到 10 条消息
  2. 5秒后自动提交 offset=10
  3. Consumer 还没来得及处理这 10 条就挂了
  4. 重启后从 offset=10 开始 → 这 10 条消息丢了！

自动提交导致重复消费:
  1. poll() 拉到 5 条消息 (offset 0~4)
  2. Consumer 处理了 3 条
  3. 自动提交 offset=5
  4. 此时 Consumer 挂了
  5. 重启后 offset=5 → 已处理的 3 条也可能重复（如果 offset 没写到 Broker）
```

### 3.3 手动提交的两种方式

| 方式 | API | 行为 | 适用 |
|------|-----|------|------|
| **commitSync()** | 同步阻塞提交 | 提交成功才返回，失败可重试 | 可靠性优先 |
| **commitAsync()** | 异步非阻塞提交 | 提交后立即返回，失败不重试（可能丢 Offset） | 性能优先 |

```java
// commitSync：阻塞等待 Broker 确认
try {
    consumer.commitSync();
} catch (CommitFailedException e) {
    log.error("提交失败", e);
    // 可以重试
}

// commitAsync：提交后立即返回，专注性能
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("异步提交失败", exception);
        // 注意：异步提交失败通常不重试（重试可能导致 offset 覆盖）
    }
});
```

> ⚠️ **关键陷阱：先提交 Offset 再处理 → 消息丢失！**
>
> ```java
> // ❌ 错误做法
> consumer.poll(); → 异步线程池处理 → 立刻同步提交 offset
> // 机器宕机 → offset 已提交但消息没处理完 → 消息丢失
>
> // ✅ 正确做法
> consumer.poll(); → 处理完所有消息 → 再提交 offset
> ```

---

## 四、auto.offset.reset —— 从哪里开始消费

| 值 | 含义 | 触发条件 |
|-----|------|---------|
| **earliest** | 从头开始 | Consumer Group 没有已提交的 Offset |
| **latest**（默认） | 从最新位置开始 | Consumer Group 没有已提交的 Offset |
| **none** | 报错 | 找不到 Offset 直接抛异常 |

```java
props.put("auto.offset.reset", "earliest");  // 新 Consumer Group 从头消费
```

> 🔴 **线上事故：** 新 Consumer Group 不小心用默认 `latest`，启动后错过了所有历史消息。如果有回溯需求，一定用 `earliest`。

---

## 五、消费者调优

### 5.1 核心参数

| 参数 | 含义 | 建议 |
|------|------|------|
| `fetch.min.bytes` | 最少拉取字节数 | 默认 1，调大可减少请求次数 |
| `fetch.max.wait.ms` | 最多等多久凑 batch | 默认 500ms |
| `max.poll.records` | 单次 poll 最多拉几条 | 默认 500，积压期可调大（如 2000） |
| `max.partition.fetch.bytes` | 单个 Partition 最多拉多少 | 默认 1MB |

### 5.2 加快积压消费的调优

```
积压清理期参数调整:
  fetch.min.bytes = 1024 * 1024     // 1MB — 每次尽量拉满
  max.poll.records = 2000            // 每次拉 2000 条（加大）
  max.partition.fetch.bytes = 5MB    // 扩大单分区拉取上限

注意: 调大 max.poll.records → 处理时间变长 → 需配合增大 max.poll.interval.ms
```

---

## 六、消息积压排错 SOP

> 凌晨 3 点报警：Topic 积压上百万条。

```
SOP（标准操作流程）:

1. 确认积压范围
   - 哪个 Consumer Group？哪个 Topic？积压量？
   - kafka-consumer-groups --describe 查看 LAG

2. 排查消费能力
   - 消费者实例数量够不够？（能否扩容）
   - 单条消息处理耗时？（是否有慢查询/慢 IO）
   - Consumer 有没有频繁 Rebalance？

3. 快速止损
   - 增加消费者实例（水平扩容）
   - 调大 max.poll.records 并降低非核心处理逻辑
   - 临时跳过非核心消息

4. 根因分析
   - 上游生产量是否突发增长？
   - 下游依赖（DB/Redis）是否有瓶颈？
   - 消费者是否有异常日志？

5. 长期方案
   - 增加 Partition 数（需提前规划）
   - 消费能力预留 3~5 倍 buffer
   - 监控告警阈值合理设置
```

---

## 七、消费失败处理

> 某条消息反复处理失败怎么办？

```
策略:
  1. 重试 N 次（如 3 次）
  2. 仍失败 → 不 requeue（避免死循环）
  3. 写入死信队列/死信 Topic
  4. 告警通知人工介入

Kafka 实现:
  - 消费失败 → 写入专门的 "dead-letter-topic"
  - 用独立消费者处理死信 → 人工排查修复后回放
```

---

> 📖 上一篇：[[05-Kafka 生产者]] | 下一篇：[[07-Kafka 存储与性能]]
