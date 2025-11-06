---
tags:
  - 八股
  - 消息队列
  - Kafka
  - Kafka生产者
status: 学习中
created: 2026-05-06
---

# Kafka 生产者机制

> Kafka 生产者是整个消息链路的起点，理解其内部机制是掌握 Kafka 的基础。

---

## 一、生产者工作流程

```
Producer 发送流程:
  ┌─────────────────────────────────────────────────────┐
  │  send(msg)                                           │
  │    ↓                                                 │
  │  Serializer（序列化 key + value）                      │
  │    ↓                                                 │
  │  Partitioner（计算目标 Partition）                    │
  │    ↓                                                 │
  │  RecordAccumulator（缓冲区，按 Partition 分组批）       │
  │    ↓                                                 │
  │  Sender 线程（批量发送）                               │
  │    ↓                                                 │
  │  Broker 响应 → Callback / Future.get()               │
  └─────────────────────────────────────────────────────┘
```

**核心组件：**

| 组件 | 作用 |
|------|------|
| **Serializer** | 将 Java 对象序列化为字节数组（支持 String、Avro、Protobuf 等） |
| **Partitioner** | 决定消息发送到哪个 Partition |
| **RecordAccumulator** | 内存缓冲区，每个 `<Topic, Partition>` 一个 Deque，消息先放这里凑批 |
| **Sender 线程** | 独立 I/O 线程，从 Accumulator 取批量消息，封装成 Request 发送到 Broker |

---

## 二、三种发送模式

| 模式 | 写法 | 可靠性 | 吞吐 | 适用场景 |
|------|------|--------|------|---------|
| **发后即忘** | `send(msg)` 不等 Future | 🔴 最低 | ⚡ 最高 | 日志、点击流 |
| **同步发送** | `send(msg).get()` 阻塞等 | 🟢 最高 | 🐢 最低 | 金融、核心通知 |
| **异步发送** | `send(msg, callback)` | 🟡 较高 | ⚡ 较高 | **大多数业务场景推荐** |

```java
// 发后即忘
producer.send(new ProducerRecord<>("topic", "key", "value"));

// 同步发送
RecordMetadata meta = producer.send(record).get();

// 异步发送（推荐）
producer.send(record, (metadata, exception) -> {
    if (exception == null) {
        log.info("发送成功: offset={}, partition={}", metadata.offset(), metadata.partition());
    } else {
        log.error("发送失败", exception);
    }
});
```

---

## 三、分区策略（Partitioner）

### 3.1 三种分区策略

| 策略 | 行为 | 特点 |
|------|------|------|
| **指定 Partition** | 直接指定 `partition` 字段 | 精确控制 |
| **有 Key** | `hash(key) % partition_count` | 相同 Key 落同一分区（局部有序） |
| **无 Key（默认）** | Sticky Partitioner（粘性分区） | 高性能 + 负载均衡 |

### 3.2 Sticky Partitioner vs Round-Robin

> Kafka 2.4+ 默认使用 **粘性分区策略**

```
Round-Robin（老版本）:
  msg1→P0, msg2→P1, msg3→P0, msg4→P1 ...
  问题：每个 Partition 只凑到 1 条消息就发 → 批量小，网络开销大

Sticky Partitioner:
  msg1→P0, msg2→P0, msg3→P0 ... batch 满 → 发给 P0
  然后换下一个 Partition → 发给 P1
  优势：同一批次消息多，网络利用率高，延迟更低
```

---

## 四、acks 参数详解

| acks | Broker 行为 | 丢数据风险 | 延迟 |
|------|------------|-----------|------|
| **0** | 不等确认，发了就跑 | 🔴 最高 | 最低 |
| **1** | Leader 写入 Page Cache 即确认 | 🟡 中等（Leader 宕机可能丢） | 中 |
| **all / -1** | 所有 ISR 副本确认 | 🟢 最低 | 最高 |

```java
// 核心链路（支付等）
props.put("acks", "all");
props.put("min.insync.replicas", 2);  // 至少 2 个 ISR 副本

// 非核心链路（日志等）
props.put("acks", "1");
```

**场景题：3 副本，min.insync.replicas=2，挂一台 Broker 还能写吗？**

> 可以。3 副本有 3 个 ISR。挂 1 台 → 剩 2 个 ISR ≥ min.insync.replicas=2 → 还能写。
> 再挂 1 台 → 只剩 1 个 ISR < 2 → **不能写**，抛出 `NotEnoughReplicasException`。

---

## 五、幂等性生产者（Idempotent Producer）

> 通过 PID + Sequence Number 实现**单分区内防重复发送**。

### 5.1 核心机制

```java
props.put("enable.idempotence", true);  // 开启幂等性
```

| 概念 | 说明 |
|------|------|
| **PID (Producer ID)** | 生产者初始化时 Broker 分配，唯一标识一个生产者实例 |
| **Sequence Number** | 对每个 `<PID, Topic, Partition>`，单调递增的序列号 |
| **Epoch** | 生产者会话版本号，用于 Zombie Fencing（僵尸生产者隔离） |

### 5.2 Broker 去重逻辑

```
Broker 为每个 <PID, Partition> 维护最近已提交的最大 seq
收到消息后:
  seq == lastSeq + 1 → 正常写入，更新 lastSeq
  seq <= lastSeq     → 重复消息，丢弃但返回 ACK
  seq > lastSeq + 1  → 有消息缺失（乱序），拒绝写入
```

> 🔑 **关键理解：** 幂等性只保证**单分区 + 单会话**内的不重复。生产者重启后 PID 变了，无法跨会话去重。

---

## 六、Kafka 事务机制

> 解决 **跨 Partition 原子写入** 和 **Consume-Transform-Produce 循环** 的 Exactly-Once。

### 6.1 核心组件

```
Producer (Transactional ID: "order-producer-1")
    │
    ├── ① initTransactions() ──→ Transaction Coordinator (TC)
    │                              │
    │                              └── 写入 __transaction_state (WAL)
    │
    ├── ② beginTransaction() ──→ TC 记录事务开始
    │
    ├── ③ send(P0, msg1) ──→ P0 Leader 收到消息（暂不可见）
    ├── ④ send(P1, msg2) ──→ P1 Leader 收到消息（暂不可见）
    │                        P0、P1 通知 TC：我参与了事务
    │
    └── ⑤ commitTransaction()
              │
              TC 类 2PC 流程：
                阶段1 PREPARE: TC 写 PREPARE_COMMIT 到 __transaction_state
                              → 向所有参与 Partition 写 Commit Marker
                阶段2 COMMIT:  TC 写 COMMITTED 到 __transaction_state
                              → 消费者通过 isolation.level=read_committed 可见
```

### 6.2 Transactional ID 的作用

| 作用 | 说明 |
|------|------|
| **跨会话恢复** | 生产者重启后，用同一 Transactional ID 可恢复未完成的事务 |
| **Zombie Fencing** | 新实例 Epoch+1，Broker 拒绝旧 Epoch 的请求，防止僵尸写入 |

### 6.3 事务主要解决什么问题？

1. **Consume-Transform-Produce 原子性：** 从 Topic A 消费 → 处理 → 写入 Topic B → 提交消费位移，全过程原子化
2. **跨 Partition 原子写入：** 一笔业务要同时写 3 个 Partition，要么全成功要么全不可见
3. **Zombie Producer 防护：** 生产者假死后新实例替代，旧实例无法继续写入

---

## 七、Exactly-Once 语义（EOS）三层架构

> 三层不是三选一，是**自下而上叠加**：开事务必须先开幂等，消费者配合 Read Committed 才看到完整效果。

```
┌─────────────────────────────────────────────────────────┐
│ 第三层: 事务 (Transactions)     ← Producer 侧             │
│  跨 Partition 原子写入，要么全成功要么全不可见              │
│  实现：类 2PC，由 Transaction Coordinator 协调            │
├─────────────────────────────────────────────────────────┤
│ 第二层: 幂等性 (Idempotence)    ← Producer 侧             │
│  单 Partition 单会话内不重复发送                          │
│  实现：PID + 递增 Sequence Number，Broker 按序去重        │
├─────────────────────────────────────────────────────────┤
│ 第一层: Read Committed 隔离     ← Consumer 侧             │
│  消费者只看已提交事务消息，未提交/已回滚的不可见            │
│  实现：isolation.level=read_committed                    │
└─────────────────────────────────────────────────────────┘
```

### 7.1 第一层：Read Committed 隔离

**一句话：消费者只读已提交的事务消息，写入中或已回滚的看不到。**

```
Producer 开启事务：
  beginTransaction()
  send P0, msg1   ← 消息已写入 P0，但标记为"未提交"
  send P1, msg2   ← 同上
  commitTransaction()  ← 提交后消息对外可见

read_uncommitted（默认）：msg1 一进 P0 就能拉走，事务未提交也能看到
read_committed：         等到 commit 后 msg1 才出现
```

> ⚠️ 不设 read_committed 的后果：事务回滚了，但消费者已经消费了回滚的消息 → 脏数据。

### 7.2 第二层：幂等性（Idempotence）

**一句话：同一条消息发两次，Broker 只存一次。**

```
原理：PID + Sequence Number

Producer 启动 → Broker 分配 PID=100
每条发往 <PID=100, Partition=0> 的消息带一个递增 seq

Broker 收到消息(PID=100, seq=5):
  seq=5 == lastSeq+1=5  → 正常写入 ✅
  seq=5 == lastSeq=5    → 重复，丢弃但返回 ACK（安抚 Producer）
  seq=5 > lastSeq+1     → 缺失/乱序，拒绝 ❌
```

**局限：** 只保证**单 Partition + 单会话**不重复。Producer 重启 → PID 变了 → 跨会话防不了。

### 7.3 第三层：事务（Transactions）

**一句话：多个 Partition 要么全写成功，要么全不可见。**

```
幂等只能防"同一条消息不重复"。
事务解决的是"两条消息要写不同 Partition，必须同时成功"。

场景：订单创建要同时写「订单 Partition」和「支付 Partition」

无事务：
  send P_订单, msg1 ✅  → 订单创建成功
  send P_支付, msg2 ❌  → 发送失败！
  → 订单创建了但支付没发起，数据不一致

有事务：
  beginTransaction()
    send P_订单, msg1
    send P_支付, msg2
  commitTransaction()
  → 要么两个都可见，要么全不可见（回滚时写 Abort Marker）
```

### 7.4 三层串联关系

```
一次跨 Partition 写入的完整保护链：

① 幂等（第二层）
   Producer 网络超时自动重试 → Broker 靠 PID+seq 识别重复 → 丢弃
   → 不会多写

② 事务（第三层）
   写 3 个 Partition，有一个失败 → 整个事务回滚
   → 已写的两个标记为 Abort，数据一致

③ Read Committed（第一层）
   消费者看不到 Abort 的消息，也看不到事务中间态
   → 不会读到脏数据
```

**一句话口诀：幂等防重发 → 事务防残缺 → Read Committed 防脏读。**

### 7.5 完整配置

```java
// Producer 端
props.put("enable.idempotence", true);           // 幂等性（第二层）
props.put("transactional.id", "tx-order-1");     // 事务 ID（第三层）

// Consumer 端
consumerProps.put("isolation.level", "read_committed");  // 只读已提交（第一层）
```

---

## 八、重试机制与顺序保证

### 8.1 重试核心参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `retries` | Integer.MAX_VALUE | 重试次数（受 delivery.timeout.ms 限） |
| `delivery.timeout.ms` | 120000 (2min) | 从 send() 到确认的最大总时间 |
| `retry.backoff.ms` | 100 | 每次重试间隔 |
| `request.timeout.ms` | 30000 | 单次请求超时 |

### 8.2 重试导致的乱序问题

```
场景: max.in.flight.requests.per.connection=5
  send msg1→P0（失败，待重试）
  send msg2→P0（成功）
  send msg3→P0（成功）
  msg1 重试成功 → 写入顺序: msg2, msg3, msg1 ❌ 乱序！
```

**解决方案：**

| 方案 | 配置 | 取舍 |
|------|------|------|
| 严格有序 | `max.in.flight.requests.per.connection=1` | 保序但吞吐极低 |
| **兼顾（推荐）** | `enable.idempotence=true`（Kafka 2.1+） | 允许 in.flight≤5 仍保序！通过 seq 去重 |

> Kafka 2.1+ 开启幂等性后，Broker 依据 Sequence Number 发现乱序立即拒绝，Sender 重新按序发送，**兼顾吞吐与顺序**。

### 8.3 TimeoutException 处理

```
异常处理策略:
  1. 可重试异常（LeaderNotAvailable、NotEnoughReplicas）:
     → 依赖 Producer 内置重试，不抛给业务代码

  2. 超时（delivery.timeout.ms 用尽）:
     → 写入本地死信表/文件，异步补偿重发

  3. 不可重试异常（RecordTooLarge、Serialization）:
     → 立即告警，修复消息本身
```

---

## 九、消息压缩

| 压缩算法 | 压缩比 | 吞吐 |
|---------|--------|------|
| **LZ4** | 中 | 🟢 最快（推荐平衡之选） |
| **ZSTD** | 🟢 最高 | 🟡 中等（带宽紧张首选） |
| GZIP | 高 | 🐢 慢 |
| Snappy | 低 | 🟢 快 |

```java
props.put("compression.type", "lz4");  // 推荐
```

> 💡 压缩在 Producer 端做，Broker 保持原样不解压（减少 CPU 开销），Consumer 端解压。

---

## 十、大消息处理

> Kafka 默认 `max.message.bytes=1MB`，超过拒绝。

**解决方案：**

```
方案一（调整参数）:
  Broker: message.max.bytes=10MB
  Producer: max.request.size=10MB
  Consumer: fetch.max.bytes=10MB
  → 不推荐：大消息拖慢整体吞吐

方案二（推荐）:
  大文件 → 存 OSS/S3 → 消息只发 URL
  → Kafka 只传输元数据，数据走对象存储

方案三:
  大消息切片 → 多条消息发送 → 消费者重组
```

---

> 📖 上一篇：[[04-Kafka 核心原理]] | 下一篇：[[06-Kafka 消费者]]
