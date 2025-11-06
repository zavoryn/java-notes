---
tags:
  - 八股
  - 消息队列
  - Kafka
status: 学习中
created: 2026-05-06
---

# Kafka 核心原理

> Kafka 本质是一个**分布式流平台**，核心是 append-only 的日志存储。强项：百万级吞吐、消息回溯。

---

## 一、Topic → Partition → Segment 三层结构

```
Topic: "user-order"（逻辑概念，像数据库的"表"）
  │
  ├── Partition 0: [msg0, msg1, msg2, msg3, ...] ← 有序日志
  │     ├── Segment: 00000000000000000000.log（1GB）
  │     ├── Segment: 00000000000000500000.log（1GB）
  │     └── Segment: 00000000000001000000.log（1GB）
  │
  ├── Partition 1: [msg0, msg1, msg2, msg3, ...]
  │     └── ...
  │
  └── Partition 2: [msg0, msg1, msg2, msg3, ...]
        └── ...
```

| 层级 | 说明 |
|------|------|
| **Topic** | 消息的逻辑分类，类似数据库的"表" |
| **Partition** | 物理分区，实现并行和水平扩展。每个 Partition 是**只追加（append-only）**的有序日志文件。同一 Partition 内严格有序，不同 Partition 间无序 |
| **Segment** | Partition 的物理文件分片，默认 1GB 一个文件，便于清理和定位 |

---

## 二、Offset 管理

```
Partition: [msg0, msg1, msg2, msg3, msg4, msg5, ...]
              ↑                    ↑
           earliest             latest

Consumer Group 消费进度:
  Consumer-1: offset=3（已消费 msg0~msg2，下一条从 msg3 开始）
  Consumer-2: offset=5（已消费 msg0~msg4，下一条从 msg5 开始）
```

**Offset 存储位置演进：**
- 老版本（< 0.9）：存在 Zookeeper
- 新版本（>= 0.9）：存在 Kafka 内部 Topic `__consumer_offsets`

```java
// 手动提交偏移量
consumer.commitSync();                    // 同步提交（阻塞）
consumer.commitAsync(callback);           // 异步提交（推荐，性能好）

// 从指定位置开始消费
consumer.seek(partition, offset);         // 跳到指定 offset
consumer.seekToBeginning(partitions);     // 从头开始
consumer.seekToEnd(partitions);           // 从最新消费
```

---

## 三、消费者组（Consumer Group）

```
Topic: order-topic (3 Partitions)
                    │
     ┌──────────────┼──────────────┐
     ↓              ↓              ↓
Partition-0    Partition-1    Partition-2
     │              │              │
     └──────┬───────┘              │
            ↓                      ↓
       Consumer-1             Consumer-2
       (消费P0+P1)            (消费P2)

         └──── 消费者组 order-group ────┘
```

**核心规则（铁律）：**
- 一个 Partition 只能被同一个消费者组内的**一个** Consumer 消费
- 一个 Consumer 可以消费**多个** Partition
- Consumer 数量 > Partition 数量 → 多出的 Consumer 空闲

---

## 四、ISR（In-Sync Replicas）机制

> Kafka 高可用的核心机制。

```
Partition-0:
  ┌────────────────────────────────────┐
  │  Leader  (Broker-1)  ← 读写都在这   │
  ├────────────────────────────────────┤
  │  Follower (Broker-2) ← 同步中 ✅    │  ← ISR（同步副本集合）
  │  Follower (Broker-3) ← 同步中 ✅    │      [Broker-1, Broker-2, Broker-3]
  │  Follower (Broker-4) ← 滞后太多 ❌   │      Broker-4 不在 ISR 中
  └────────────────────────────────────┘
```

| 概念 | 说明 |
|------|------|
| **Leader** | 负责所有读写，每个 Partition 只有一个 Leader |
| **Follower** | 从 Leader 同步数据，不对外服务 |
| **ISR** | Leader + 跟得上的 Follower 集合 |

### acks 参数三档

| acks 值 | 含义 | 安全性 | 性能 |
|---------|------|--------|------|
| **acks=0** | 不等确认，发了就跑 | 🔴 最不安全 | ⚡ 最快 |
| **acks=1** | Leader 写入成功即确认 | 🟡 中等 | ⚡ 较快 |
| **acks=all / -1** | 所有 ISR 写入成功才确认 | 🟢 最安全 | 🐢 最慢 |

### min.insync.replicas 配合 acks=all

```java
Properties props = new Properties();
props.put("acks", "all");               // 所有 ISR 确认
props.put("min.insync.replicas", 2);    // 至少 2 个副本同步
```

> 💡 **含义：** acks=all 要求所有 ISR 确认，min.insync.replicas=2 要求 ISR 至少包含 2 个 Broker。两者配合，Leader + 至少 1 个 Follower 写入成功才返回。

---

## 五、Kafka 为什么能做到高吞吐？

1. **磁盘顺序写入：** append-only 日志，顺序写比随机写快 6000 倍，媲美内存
2. **Page Cache（页缓存）：** 利用 OS 的页缓存，避免 JVM 堆内缓存（GC 问题）
3. **零拷贝（Zero Copy）：** `sendfile()` 系统调用，数据从磁盘直接到网卡，绕过用户态
4. **批量处理：** Producer 批量发送、Consumer 批量拉取，减少网络往返
5. **分区并行：** Topic 分多个 Partition，多 Broker 并行读写
6. **消息压缩：** 支持 LZ4、ZSTD、GZIP 等压缩，减少网络传输

---

> 📖 上一篇：[[03-消息队列通用问题]] | 下一篇：[[05-Kafka 生产者]] | [[06-Kafka 消费者]]
