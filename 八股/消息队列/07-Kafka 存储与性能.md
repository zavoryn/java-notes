---
tags:
  - 八股
  - 消息队列
  - Kafka
  - Kafka存储
status: 学习中
created: 2026-05-06
---

# Kafka 存储与性能

> Kafka 百万级吞吐的秘密——全在这一章。核心技术：磁盘顺序写入、Page Cache、零拷贝。

---

## 一、文件存储结构

### 1.1 三层物理结构

```
Topic: "user-order"
  │
  ├── Partition 0/
  │     ├── 00000000000000000000.log        ← 日志文件（消息数据）
  │     ├── 00000000000000000000.index      ← 稀疏索引（offset→位置）
  │     ├── 00000000000000000000.timeindex  ← 时间索引
  │     ├── 00000000000000100000.log        ← 第二段 1GB
  │     ├── 00000000000000100000.index
  │     └── ...
  │
  └── Partition 1/
        └── ...
```

### 1.2 Log Segment 详解

| 文件 | 作用 |
|------|------|
| **`.log`** | 消息实际存储，append-only，默认 1GB 滚动 |
| **`.index`** | 稀疏索引：记录 offset ↔ 物理位置的映射（不是每条消息都有） |
| **`.timeindex`** | 时间戳索引：按时间查找消息 |

### 1.3 通过 Offset 定位消息

```
查找 offset=1025 的消息:
  1. 二分查找所有 Segment 文件名 → 定位到 Segment 00000000000000001000.log
  2. 在该 Segment 的 .index 中二分查找 ≤1025 的最大索引条目
  3. 获取物理位置 position
  4. 从 .log 文件的 position 开始顺序扫描，直到找到 offset=1025
```

> 💡 稀疏索引 + 顺序扫描 = 高效定位。索引每 4KB 记录一条（约 500 条消息记录一条索引）。

---

## 二、Message 结构

```
一条 Kafka Message（V2 格式）:
┌────────────────────────────────────┐
│  Offset（8 字节）                    │
│  Length（4 字节）                    │
│  Leader Epoch（4 字节）              │
│  Magic（1 字节，V2=2）               │
│  CRC（4 字节）                       │
│  Attributes（2 字节，压缩类型等）     │
│  Timestamp（8 字节）                 │
│  Key Length + Key（可选）            │
│  Value Length + Value（消息体）      │
│  Headers（可选元数据）               │
└────────────────────────────────────┘
```

> V2 格式（Kafka 0.11+）支持 Record Batch，将多条消息打包在同一批次中，共享头和压缩，大幅提升吞吐。

---

## 三、Page Cache（页缓存）机制

> Kafka 依赖 **OS 的 Page Cache** 而非 JVM 堆内缓存，这是高性能的关键设计。

### 3.1 为什么不用 JVM 堆缓存？

```
JVM 堆缓存问题:
  1. GC 压力大 — 缓存越多，GC 越久
  2. 对象开销大 — Java 对象 40+ 字节 header
  3. 堆外内存管理复杂

Page Cache 优势:
  1. OS 管理，完全避开 JVM GC
  2. 所有空闲内存自动做 Page Cache
  3. 读写都在 Page Cache 中完成（热数据不走磁盘）
```

### 3.2 工作流程

```
写入: Producer → 网卡 → Page Cache → 异步刷盘
读取: Consumer ← 网卡 ← Page Cache ← 磁盘（仅冷数据）

大部分读写命中 Page Cache → 相当于读写内存，极快
```

### 3.3 断电数据丢失问题

> Broker 服务器意外断电，Page Cache 中未刷入磁盘的数据会丢吗？

```
答案: 会丢失！Page Cache 是内存，断电即失。
但影响有限，因为:
  1. Kafka 有副本机制 — 其他 Broker 副本上有数据
  2. 未 flush 的消息量通常很小（fsync 间隔短）
  3. acks=all + min.insync.replicas ≥ 2 → 至少 2 台 Broker 有数据

配置:
  log.flush.interval.messages   — 多少条消息触发 fsync
  log.flush.interval.ms         — 多久触发 fsync
  → 默认由 OS 决定，通常不做强制同步刷新（牺牲一点安全换性能）
```

---

## 四、零拷贝（Zero Copy）

> Kafka 消费高性能的核心技术。

### 4.1 传统读取 vs 零拷贝

```
传统 read/write:
  磁盘 → 内核缓冲区 → 用户缓冲区 → Socket 缓冲区 → 网卡
  4 次上下文切换 + 2 次 CPU 拷贝 + 2 次 DMA 拷贝

零拷贝 (sendfile):
  磁盘 → 内核缓冲区 ──→ Socket 缓冲区 → 网卡
                 (直接传递，不走用户态)
  2 次上下文切换 + 0 次 CPU 拷贝 + 2 次 DMA 拷贝
```

**节省了：** 不经过用户态 → 减少 CPU 拷贝 → 减少上下文切换 → 吞吐提升数倍

```java
// Kafka 内部使用 FileChannel.transferTo() 实现零拷贝
// 底层调用 Linux sendfile() 系统调用
```

---

## 五、日志清理策略

| 策略 | 参数 | 行为 |
|------|------|------|
| **按时间删除** | `log.retention.hours=168`（7 天） | 超时 Segment 直接删除 |
| **按大小删除** | `log.retention.bytes=-1`（无限制） | 总日志超此大小清理旧 Segment |
| **Compact 压缩** | `log.cleanup.policy=compact` | 保留每个 Key 的最新值（类似 LSM 树） |

### Compact 策略详解

```
Compact 前:
  key:A → value:1
  key:B → value:2
  key:A → value:3  ← A 的最新值
  key:B → value:4  ← B 的最新值

Compact 后:
  key:A → value:3
  key:B → value:4
  (旧值被清理)
```

> Compact 适用于：数据库变更日志（CDC）、KTable 状态存储。

---

## 六、HW 和 LEO

> 面试高频考点，理解这两个概念才能懂 Kafka 副本同步。

### 6.1 定义

```
Partition:
  [msg0, msg1, msg2, msg3, msg4, msg5, ...]
           ↑              ↑
          HW            LEO

  LEO (Log End Offset): 下一条要写入消息的 Offset（已写入的最后一条 offset+1）
  HW  (High Watermark): 所有 ISR 副本都已完成同步的最大 Offset
                        → Consumer 最多能看到 HW 之前的消息
```

### 6.2 为什么需要 HW？

```
场景: acks=all 模式
  Leader:     [msg0, msg1, msg2, msg3]  LEO=4
  Follower-1: [msg0, msg1]              LEO=2
  Follower-2: [msg0, msg1, msg2]        LEO=3

  HW = min(ISR 中所有 LEO) = min(4, 2, 3) = 2
  → Consumer 只能消费到 offset=2（msg0, msg1）
  → 保证 Consumer 只读到所有副本都已确认的消息
```

---

## 七、Kafka 高吞吐总结

| 机制 | 原理 | 效果 |
|------|------|------|
| **磁盘顺序写入** | append-only 日志，顺序写比随机写快 6000 倍 | I/O 不成为瓶颈 |
| **Page Cache** | 利用 OS 页缓存，避免 JVM GC | 读写接近内存速度 |
| **零拷贝** | `sendfile()` 绕过用户态 | 消费端吞吐 ↑ 数倍 |
| **批量发送** | Producer buffer + Sender 线程批量发送 | 减少网络往返 |
| **批量拉取** | Consumer 一次 pull 多条消息 | 减少网络请求 |
| **消息压缩** | Producer 端 LZ4/ZSTD 压缩 | 减少网络带宽占用 |
| **分区并行** | 多 Partition 多 Broker 并行 | 水平扩展线性增长 |

---

> 📖 上一篇：[[06-Kafka 消费者]] | 下一篇：[[08-Kafka 集群与运维]]
