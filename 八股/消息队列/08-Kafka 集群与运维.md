---
tags:
  - 八股
  - 消息队列
  - Kafka
  - Kafka运维
status: 学习中
created: 2026-05-06
---

# Kafka 集群与运维

> 集群管理和运维是 Kafka 面试中的高级话题，考察你对 Kafka 全局架构的理解深度。

---

## 一、Controller（控制器）

### 1.1 Controller 职责

> Controller 是 Kafka 集群的"大脑"，负责全局管理。

```
Controller 的核心职责:
  1. Partition Leader 选举
     - 某个 Broker 挂了 → Controller 为受影响的 Partition 选新 Leader
  2. 集群元数据管理
     - Topic 创建/删除、Partition 增加
  3. ISR 变更通知
     - Follower 掉队/追上 → 更新 ISR → 广播到集群
  4. 分区再分配 (Reassign)
     - 运维操作：将 Partition 迁移到其他 Broker
```

### 1.2 Controller 选举

```
集群中只有一个 Controller，由 Zookeeper 选出:
  - 所有 Broker 争抢 ZK 节点 /controller
  - 先创建成功的成为 Controller
  - 当前 Controller 挂掉 → ZK 临时节点消失 → 其他 Broker 竞争新 Controller

KRaft 模式变更:
  - Controller 选举不再依赖 ZK
  - 通过 KRaft 共识协议（类 Raft）内部选举
  - Controller 数量从 1 个变成可配置的 Quorum
```

### 1.3 Controller 宕机影响

> Controller 挂掉后，集群短期内无法处理管理操作，但**已存在的读写不受影响**。

```
Controller 宕机 → 发生什么？
  1. ZK /controller 临时节点超时消失
  2. 其他 Broker 竞争成为新 Controller
  3. 新 Controller 从 ZK 读取集群元数据
  4. 重新恢复管理能力

这期间:
  - 已有 Partition 的读写正常（Producer/Consumer 直连 Leader）
  - 无法创建 Topic、无法 Partition 重分配
  - 无法处理新的 Broker 宕机导致的 Leader 选举
```

---

## 二、Leader 选举机制

### 2.1 选举策略

> Kafka 的 Leader 选举不是在 Follower 之间选，而是 **Controller 直接指定**。

```
Leader 选举优先级:
  1. 首选 AR (Assigned Replicas) 中在 ISR 内的副本
  2. 优先选择 Preferred Leader（首个副本/AR 中的第一个）
```

### 2.2 为什么不直接在 Follower 间投票？

```
Kafka 的设计权衡:
  - ZK / KRaft 已维护了全局元数据
  - Controller 知道所有副本的同步状态（ISR）
  - 直接指定 Leader → 无投票开销 → 快速恢复

对比 Raft:
  - Raft: Follower 之间投票选 Leader
  - Kafka: Controller 直接指定（更快，但依赖 Controller 可用）
```

---

## 三、ISR 边界场景题

### Q：3 副本，min.insync.replicas=2，陆续宕机。

```
初始: Broker-1 (Leader), Broker-2, Broker-3 → ISR=[1,2,3]

挂 Broker-3:
  ISR=[1,2], min.insync.replicas=2 → 2 ≥ 2 → 还能写 ✅

再挂 Broker-2:
  ISR=[1], min.insync.replicas=2 → 1 < 2 → 不能写了 ❌
  抛出 NotEnoughReplicasException
```

> 💡 这就是为什么 **核心链路 acks=all + min.insync.replicas=2**，同时要确保副本数 ≥ min.insync.replicas + 可容忍的故障数。

---

## 四、Partition 迁移

### Q：单台 Broker 磁盘快满了，如何不停机迁移 Partition？

```
在线迁移步骤:
  1. 准备 JSON 迁移方案文件:
     {
       "partitions": [
         {"topic": "order-topic", "partition": 0, "replicas": [2,3,4]}
       ],
       "version": 1
     }

  2. 执行迁移:
     kafka-reassign-partitions --bootstrap-server localhost:9092 \
       --reassignment-json-file reassign.json \
       --execute

  3. 查看进度:
     kafka-reassign-partitions --bootstrap-server localhost:9092 \
       --reassignment-json-file reassign.json \
       --verify

  迁移过程:
    - Broker 间复制数据（Follower 变 Leader 的过程）
    - 对生产和消费无影响（Leader 切换时短暂停顿）
```

---

## 五、Partition 数量规划

### 5.1 分区过少/过多的影响

```
分区太少:
  - 并发度低，吞吐无法扩展
  - 单个 Partition 数据太大（磁盘、清理慢）
  - Consumer 实例数受限于 Partition 数

分区太多:
  - 每个 Partition 占用文件句柄、内存
  - Leader 选举、Controller 管理开销增大
  - 延迟可能增加（更多 Partition 竞争 I/O）
  - 极端案例: 上万 Topic → Controller 负载过高 → 集群不稳定
```

### 5.2 分区数规划公式

```
Partition数 = max(目标吞吐 / 单分区吞吐, 目标并发消费者数)

经验值:
  - 单分区吞吐: 10~50 MB/s（取决于硬件）
  - 建议总 Partition 数 < Broker 数 × 2000（粗糙上限）
  - 单 Broker 建议 2000~4000 Partition 以内
```

---

## 六、KRaft 模式（去 Zookeeper）

> Kafka 2.8+ 引入 KRaft，3.5+ 生产可用，不再依赖 Zookeeper。

### 6.1 为什么去 ZK？

```
Zookeeper 的问题:
  1. 额外的组件，运维复杂
  2. ZK 和 Kafka 元数据一致性保障困难
  3. ZK 本身有性能瓶颈（所有写走 Leader）
  4. 两套存储、两套监控

KRaft 优势:
  1. 单一进程，简化部署
  2. 元数据存储在 Kafka 内部 Topic @metadata
  3. 元数据操作性能更高（无 ZK 往返）
  4. Controller Quorum 实现高可用（类似 Raft）
```

### 6.2 KRaft 架构

```
KRaft 两种角色:
  Controller 节点: 管理元数据（替代 ZK），形成 Quorum
  Broker 节点: 处理消息读写

混合模式: 节点可同时是 Controller + Broker
专用模式: 单独部署 Controller 节点（生产推荐）

Quorum: 3 或 5 个 Controller 节点，多数派决策
```

---

## 七、多租户与 Quota 机制

### Q：如何限制某业务线过度使用带宽？

```bash
# 限制 Producer 流量
kafka-configs --alter --add-config 'producer_byte_rate=1048576' \
  --entity-type clients --entity-name clientA

# 限制 Consumer 流量
kafka-configs --alter --add-config 'consumer_byte_rate=2097152' \
  --entity-type clients --entity-name clientA

# 限制请求速率
kafka-configs --alter --add-config 'request_percentage=50' \
  --entity-type users --entity-name userA
```

---

## 八、JVM 堆内存配置

> 为什么物理机内存充裕，但 Kafka 堆内存只设 6~8GB？

```
原因:
  1. Kafka 用大量 OS Page Cache，而不是 JVM 堆
     → JVM 堆小 = GC 压力小 = 停顿短
     → 剩余内存全给 Page Cache（OS 自动管理）

  2. 堆内存最佳实践:
     物理内存 32GB → JVM 堆 6GB, Page Cache ~26GB
     物理内存 64GB → JVM 堆 8GB, Page Cache ~56GB

  3. 配置:
     KAFKA_HEAP_OPTS="-Xmx6g -Xms6g"

  4. 32GB 以上堆注意: JVM 指针压缩在 32GB 以下，超过会变慢
```

---

## 九、网络模型

```
Kafka Broker 网络线程模型:
  ┌─────────────────────────────────────┐
  │  Acceptor Thread（接收新连接）         │
  │    ↓                                │
  │  Processor Threads (Network Threads) │
  │    - 读取请求、放入 Request Queue     │
  │    ↓                                │
  │  I/O Threads (Request Handler)      │
  │    - 从 Request Queue 取请求          │
  │    - 处理（写日志、查索引）            │
  │    - 响应放入 Response Queue          │
  │    ↓                                │
  │  Processor Threads                  │
  │    - 发送响应到客户端                 │
  └─────────────────────────────────────┘

核心参数:
  num.network.threads    — 默认 3，负责网络读写
  num.io.threads         — 默认 8，负责请求处理
```

---

## 十、分区数据倾斜处理

### 问题表现

```
Partition-0: 10GB, 100万条消息
Partition-1: 100MB, 1万条消息
→ 严重倾斜！消费者负载不均
```

### 解决方案

| 方案 | 做法 |
|------|------|
| **优化 Key 设计** | 避免热点 Key（如某个 userId 特别活跃） |
| **添加随机后缀** | `key = userId + "_" + random(1,10)` 打散热点 |
| **自定义 Partitioner** | 实现特殊的分区逻辑 |
| **增加分区数** | `kafka-topics --alter --partitions 20`（只能增加不能减少！） |

> ⚠️ **注意：** Kafka 不支持减少 Partition 数！只能增加。分区数规划要慎重。

---

> 📖 上一篇：[[07-Kafka 存储与性能]] | 下一篇：[[09-消息队列选型与串联]]
