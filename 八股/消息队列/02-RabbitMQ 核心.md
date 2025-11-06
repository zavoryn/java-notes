---
tags:
  - 八股
  - 消息队列
  - RabbitMQ
status: 已掌握
created: 2026-05-06
review_date: 2026-05-11
importance: P0
---

# RabbitMQ 核心

> RabbitMQ 是 AMQP 协议的标准实现，核心能力在于 **灵活的路由** 和 **微秒级低延迟**。面试三大必考：四种 Exchange、消息不丢、死信/延时队列。

---

## 一、整体架构

```
┌────────────────── RabbitMQ Broker ────────────────┐
│                                                    │
│  ┌─────────── Virtual Host: /prod ────────────┐   │
│  │                                              │   │
│  │  Producer ──→ Exchange ──→ Queue ──→ Consumer│   │
│  │                (路由)       (存储)     (消费)   │   │
│  │                                              │   │
│  │  Producer ──→ Exchange ──→ Queue ──→ Consumer│   │
│  │                                              │   │
│  └──────────────────────────────────────────────┘   │
│                                                    │
│  ┌─────────── Virtual Host: /dev ─────────────┐    │
│  │  ...（隔离的开发环境）                         │    │
│  └──────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────┘
```

### Virtual Host（vhost）

> 每一个 vhost 就是一个"迷你 RabbitMQ"，拥有独立的 Exchange、Queue、权限。

| 特性 | 说明 |
|------|------|
| **隔离级别** | 完全隔离，互不影响 |
| **权限控制** | 每个用户对不同 vhost 有不同权限 |
| **命名空间** | 同一 vhost 内 Exchange/Queue 名称唯一 |
| **典型用法** | `/prod` 生产环境、`/dev` 开发环境、按业务线分 vhost |

### Channel（通道）

> AMQP 协议的核心设计：一条 TCP 连接上可以创建多个 Channel，复用连接资源。

```
Application
    └── Connection（一条 TCP 连接）
          ├── Channel 1（线程1 使用）
          ├── Channel 2（线程2 使用）
          ├── Channel 3（线程3 使用）
          └── ...

为什么不用多 Connection？
→ 减少 TCP 握手开销，一个 Connection 可复用成千上百 Channel
→ 类似 HTTP/2 的多路复用思想
```

**Connection vs Channel 深究（面试加分）：**

| | Connection（连接） | Channel（通道） |
|---|---|---|
| **本质** | TCP 长连接 | 虚拟连接，复用 TCP |
| **创建成本** | 高（三次握手 + AMQP 协议协商） | 低（在已建立的 TCP 连接上开辟逻辑通道） |
| **资源占用** | 一个 TCP 连接占用一个文件描述符 | 几乎无额外网络开销 |
| **线程安全** | 无要求 | **非线程安全**，一个 Channel 不能被多线程同时使用 |
| **故障隔离** | Connection 断开 → 所有 Channel 失效 | 单个 Channel 关闭不影响同 Connection 的其他 Channel |

```
最佳实践：
  一个应用进程 = 一个 Connection
  一个业务线程 = 一个 Channel（用完即关，或线程本地复用）
  不要在线程间共享 Channel ❌
```

**面试中"Channel 为什么设计成非线程安全"的加分回答：**
> "AMQP 协议设计中，Channel 上传输的命令需要严格有序（如 confirm 的 deliveryTag 递增）。如果多线程共用一个 Channel，消息顺序会被打乱，Confirm 回执会对不上。所以 RabbitMQ 官方建议一个线程一个 Channel，让开发者自己保证顺序。"

### Virtual Host 补充

---

## 二、Exchange 四种类型（核心重点！）

> Exchange 的核心工作：拿到消息后，根据 **Exchange 类型 + Binding Key + Routing Key** 决定把消息丢给哪些 Queue。

### （1）Direct Exchange（直接交换机）

- **规则：** Routing Key **完全匹配** Binding Key
- **比喻：** 快递精确投递——"收货人精确匹配才派送"

```
Exchange: order_exchange (type=direct)

Binding:  order_exchange ──("order.create")──→ order_queue
Binding:  order_exchange ──("order.cancel")──→ cancel_queue

发送消息 routingKey="order.create" → 只进 order_queue
发送消息 routingKey="order.cancel" → 只进 cancel_queue
```

### （2）Topic Exchange（主题交换机）

- **规则：** Routing Key 模式匹配
  - `*` 匹配**恰好一个**单词
  - `#` 匹配**零个或多个**单词
- **比喻：** 邮件订阅——"所有体育新闻都发给我"

```
Exchange: log_exchange (type=topic)

Binding: log_exchange ──("order.*")────→ order_log_queue
Binding: log_exchange ──("*.error")────→ error_log_queue
Binding: log_exchange ──("#")──────────→ all_log_queue

发送 "order.create" → order_log_queue ✅ all_log_queue ✅
发送 "order.create.error" → 只进 all_log_queue ✅（"order.*" 只匹配两级）
发送 "payment.error" → error_log_queue ✅ all_log_queue ✅
```

> 🔑 **记忆技巧：** `*` 是"一层楼"，`#` 是"整栋楼"。

### （3）Fanout Exchange（广播交换机）

- **规则：** 忽略 Routing Key，广播到所有绑定的队列
- **比喻：** 村口大喇叭——"所有人都能听到"

```
Exchange: broadcast_exchange (type=fanout)

Binding: broadcast_exchange ──→ sms_queue
Binding: broadcast_exchange ──→ email_queue
Binding: broadcast_exchange ──→ push_queue

发送任意消息 → 同时进入 sms_queue + email_queue + push_queue
```

**典型场景：** 订单支付成功 → 同时通知发短信、发邮件、发Push、更新积分

### （4）Headers Exchange（头交换机）

- **规则：** 根据消息 Header 属性匹配（几乎不用，了解即可）
- 支持 `x-match: all`（全匹配）和 `x-match: any`（任一匹配）

### 四种 Exchange 一表对比

| 类型 | 路由规则 | Routing Key | 典型场景 |
|------|---------|-------------|----------|
| **Direct** | 精确匹配 | 必须相等 | 任务分发：`"order.create"` → 订单服务 |
| **Topic** | 模式匹配 `*` / `#` | 点号分隔的多级词 | 日志路由：`"order.*.error"` → 订单错误日志 |
| **Fanout** | 忽略，全广播 | 不需要 | 配置更新广播：所有节点同时收到 |
| **Headers** | Header 属性匹配 | 不需要 | 极少用，被 Topic 替代 |

### ⚠️ 面试陷阱：Binding 时 Routing Key 是空的？

```
// 注意：Fanout 类型下，绑定队列时 routingKey 通常设为 ""
channel.queueBind(queueName, EXCHANGE_NAME, "");  // Fanout 忽略 Routing Key
channel.queueBind(queueName, EXCHANGE_NAME, "order.create");  // Direct/Topic 必须精确/模式匹配
```

> 🔑 面试官可能问："给 Fanout Exchange 绑定时，Routing Key 传了什么？" → **答：传了也没用，Fanout 直接忽略。但通常为了代码统一，会传空字符串 `""`。**

### 🔗 Kafka vs RabbitMQ 路由对比

| | RabbitMQ | Kafka |
|---|----------|-------|
| **路由位置** | Broker 侧（Exchange 做路由判断） | Producer 侧（生产者决定发到哪个 Partition） |
| **路由灵活性** | **极高**（四种 Exchange + 任意多 Binding） | 低（仅按 key 哈希或自定义 Partitioner） |
| **适用场景** | 复杂路由需求（如按业务类型+优先级双维度） | 顺序写入 + 高吞吐 |

> Kafka 不是没有路由，而是把路由逻辑**前置到了 Producer**——消费端不需要复杂路由，因为 Kafka 是 append-only 日志。

---

## 三、消息流转完整链路

```
1. Producer 创建 Connection → 创建 Channel
2. Producer 声明 Exchange（不存在则创建）
3. Producer 声明 Queue（不存在则创建）
4. Producer 通过 Binding 绑定 Exchange 和 Queue
5. Producer 发送消息到 Exchange（携带 Routing Key）
6. Exchange 根据规则投递消息到对应 Queue
7. Consumer 监听 Queue，收到消息后处理
8. Consumer 发送 ACK 确认消费
9. Broker 收到 ACK 后删除消息
```

---

## 四、消息可靠性（面试核心）

> "消息丢了怎么办？"——面试官必问题。

### 4.1 消息丢失的三个阶段

```
Producer ──→① Broker ──→② Consumer
                 ↓③
              磁盘/内存
```

| 阶段 | 丢消息场景 | 解决方案 |
|------|-----------|---------|
| ① 生产阶段 | 网络故障，Producer 以为发出去了 | **Publisher Confirm** |
| ② 存储阶段 | Broker 挂了，内存消息丢失 | **消息持久化** + **队列持久化** |
| ③ 消费阶段 | Consumer 拿到消息还没处理就挂了 | **手动 ACK** |

### 4.2 生产者确认：Publisher Confirm（三种模式）

```java
// 开启 Confirm 模式
channel.confirmSelect();

// 模式一：单条同步确认（性能最差 ❌）
channel.basicPublish(...);
if (channel.waitForConfirms()) { /* 成功 */ }

// 模式二：批量同步确认（平衡方案）
for (int i = 0; i < batchSize; i++) {
    channel.basicPublish(...);
}
channel.waitForConfirms(); // 等一批全部确认

// 模式三：异步确认（推荐 ✅，性能最好）
channel.addConfirmListener(new ConfirmListener() {
    public void handleAck(long deliveryTag, boolean multiple) {
        // Broker 确认收到，清理重发缓存
        if (multiple) {
            // 批量确认：<= deliveryTag 的消息全部确认
            cleanCacheUpTo(deliveryTag);
        } else {
            // 单条确认
            cleanCache(deliveryTag);
        }
    }
    public void handleNack(long deliveryTag, boolean multiple) {
        // Broker 未确认 → 重发！
        retryMessages(deliveryTag, multiple);
    }
});
```

| Confirm 模式 | 吞吐量 | 延迟 | 可靠性 | 适用场景 |
|-------------|--------|------|--------|----------|
| 单条同步 | 低（~2000 msg/s） | 高（RTT） | 高 | 对可靠性要求极高的金融场景 |
| 批量同步 | 中（~20000 msg/s） | 中 | 中 | 一般业务 |
| **异步** | **高（~100000 msg/s）** | **低** | **高** | **推荐方案** ✅ |

> 🔑 **面试金句：** "Confirm 的三种模式本质是'可靠性'和'吞吐量'的平衡——异步模式通过回调机制实现了'既要又要'。"

### 4.3 消息持久化 — 深入陷阱

```java
// 1. 队列持久化
boolean durable = true;
channel.queueDeclare(QUEUE_NAME, durable, false, false, null);

// 2. 消息持久化
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
    .deliveryMode(2)  // 2 = 持久化, 1 = 非持久化
    .build();
```

| 持久化策略 | 队列设置 | 消息设置 | 重启后 |
|-----------|---------|---------|--------|
| 都不持久 | durable=false | deliveryMode=1 | **全丢** |
| 仅队持久 | durable=true | deliveryMode=1 | 队列在，消息丢 |
| 仅消息持久 | durable=false | deliveryMode=2 | 队列没了，消息也无法恢复 |
| **全持久** | **durable=true** | **deliveryMode=2** | **两者都在 ✅** |

> **注意：** 持久化不是绝对安全！消息刚写入还没 fsync 到磁盘时 Broker 挂了也可能丢。极端场景用**镜像队列**。

**持久化的 fsync 陷阱（面试加分）：**

```
Producer 发消息 → Broker 写入 Page Cache（内存）→ 返回 ACK → 异步刷盘

问题：如果 Broker 在 Page Cache 刷盘前宕机 → 消息丢失！

解决方案：
  1. 镜像队列（推荐 ✅）：消息同步到多节点内存，不是单点
  2. 强制 fsync：性能极差（~100 msg/s），不推荐
  3. 接受极小丢失概率：配合 Confirm + 重试
```

### 4.3+ 镜像队列（Quorum Queue）

> RabbitMQ 3.8+ 引入 **Quorum Queue**（基于 Raft），逐步替代传统镜像队列。

```
                     ┌──────────────┐
    Producer ───────→│  Leader Node  │← 写入
                     └──────┬───────┘
                            │ Raft 复制
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │Follower 1│ │Follower 2│ │Follower 3│
       └──────────┘ └──────────┘ └──────────┘

       过半节点确认 → 消息才被认为"已提交"
       任意节点宕机 → 自动选举新 Leader
```

| 特性 | 传统镜像队列 | Quorum Queue（3.8+） |
|------|-------------|---------------------|
| 一致性协议 | GM（非标准，有脑裂风险） | **Raft**（业界标准） |
| 数据安全 | 可能丢消息（异步复制） | **过半确认才提交** |
| 性能 | 较高 | 略低（Raft 有开销） |
| 推荐 | 逐步淘汰 | **推荐使用** ✅ |

**数据安全与性能的最终选择：**

| 方案 | 安全性 | 性能 | 推荐场景 |
|------|--------|------|----------|
| 持久化 + Confirm + ACK | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 一般业务（99.99% 可靠） |
| 持久化 + Confirm + ACK + 镜像 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 金融/支付等强安全场景 |

### 4.4+ Consumer Prefetch（QoS 限流）

> ⚠️ 这是你笔记缺的一个关键点！面试常问："消费者处理速度不一致怎么办？"

```java
// Qos: 告诉 Broker，我一个消费者最多"预取"多少条未 ACK 消息
// prefetchSize: 消息大小限制（一般填 0，不限）
// prefetchCount: 最多同时处理的消息数
// global: false=当前Channel, true=整个Connection
channel.basicQos(0, 1, false);   // 每次只取1条，处理完ACK再拿
channel.basicQos(0, 50, false);  // 最多预取50条（批量处理场景）
```

**prefetch = 1 的含义（经典面试题）：**
> "prefetch=1 意味着消费者'一次只处理一条，ACK 后才能拿新消息'。这本质上是 **RabbitMQ 端的负载均衡机制**——处理慢的消费者拿得少，处理快的拿得多，天然实现'能者多劳'。"

```
prefetch=1:
  Consumer A（快）：取→处理→ACK→取→处理→ACK...（循环快）
  Consumer B（慢）：取→处理...（好久）...ACK→取（等很久才拿一条）

  结果：A 处理的量 >> B 处理的量，自动负载均衡 ✅

prefetch=50:
  Consumer A: 一次性拿走50条慢慢处理
  Consumer B: 一次性拿走50条慢慢处理

  问题：慢的占着50条处理不完，快的干着急 → 负载不均衡 ❌
```

**prefetch 设置建议：**
- `prefetch=1`：对延迟敏感（每条消息处理快，追求公平分发）
- `prefetch=10~50`：批量处理（如批量写库，减少网络往返）
- `prefetch=0`：无限制（⚠️ 危险！可能把所有消息拉到内存 OOM）

### 4.4 消费者 ACK（手动确认）

```java
// ❌ 自动ACK——消费即删除，处理失败消息已丢
channel.basicConsume(QUEUE_NAME, true, consumer);

// ✅ 手动ACK——处理成功再确认
channel.basicConsume(QUEUE_NAME, false, consumer);

// 处理成功 → basicAck
// 处理失败 → basicNack (requeue=true 重新入队)
```

| 方法 | 含义 | 支持批量 |
|------|------|---------|
| `basicAck` | 确认消费 ✅ | ✅ |
| `basicNack` | 拒绝 ❌ | ✅ |
| `basicReject` | 拒绝（单条）❌ | ❌ |

### 4.5 消费幂等性

> "幂等性"：同一条消息消费 1 次和消费 100 次，业务结果完全一样。

#### 方案一：数据库唯一约束

```sql
INSERT INTO message_consumed_record (msg_id, consumer, create_time)
VALUES ('msg_123456', 'order-service', NOW());
-- 重复消费时 INSERT 失败（唯一约束），跳过即可
```

#### 方案二：Redis 记录已消费消息 ID

```java
String msgId = getMessageId(message);
Boolean success = redisTemplate.opsForValue()
    .setIfAbsent("consumed:" + msgId, "1", Duration.ofHours(24));

if (Boolean.FALSE.equals(success)) {
    // 已经消费过，跳过
    channel.basicAck(deliveryTag, false);
    return;
}
processMessage(message);
channel.basicAck(deliveryTag, false);
```

---

## 五、死信队列（DLX / Dead Letter Exchange）

### 5.1 什么是死信

消息变成"死信"的三种情况：

| 情况 | 说明 | 场景 |
|------|------|------|
| **消息被拒绝** | `basicReject/Nack` 且 `requeue=false` | 消息格式错误，无法处理 |
| **消息过期** | TTL（Time To Live）到期 | 超时订单自动取消 |
| **队列满了** | 队列达到最大长度 | 系统故障，消息堆积 |

### 5.2 死信队列架构

```
                    ┌──────────┐
Producer ──────────→│ 正常Queue │──────────→ Consumer
                    └────┬─────┘
                         │ 消息过期/被拒/队列满
                         ↓
                    ┌──────────┐      ┌──────────┐
                    │   DLX    │─────→│ 死信Queue │
                    └──────────┘      └────┬─────┘
                                           │
                                           ↓
                                    ┌──────────────┐
                                    │ 告警/人工处理  │
                                    │ 或自动重试   │
                                    └──────────────┘
```

### 5.3 死信队列的实际用途

1. **延迟队列：** 利用消息 TTL + DLX 实现延时消费（30分钟后检查支付状态）
2. **异常消息处理：** 消息格式错误、业务处理失败的消息单独存储分析
3. **监控告警：** 死信队列堆积 → 触发告警 → 人工介入

### 5.4 延时队列完整设计（面试重点！）

> 场景："下单后 30 分钟未支付，自动取消订单。"

**RabbitMQ 实现方案：利用 TTL + DLX**

```
                      ┌─────────────────┐
Producer ────────────→│ delay_exchange   │
  order.create         │ (direct)        │
  expire=1800000ms     └────────┬────────┘
                                │ binding: "order.delay"
                                ↓
                      ┌─────────────────┐
                      │ delay_queue      │
                      │ TTL=30min        │  ← 消息在这里"等待"
                      │ dead_letter=     │
                      │   actual_exchange│
                      └────────┬────────┘
                               │ 30分钟后过期 → 自动转到 DLX
                               ↓
                      ┌─────────────────┐
                      │ actual_exchange  │
                      │ (direct)        │
                      └────────┬────────┘
                               │ binding: "order.cancel"
                               ↓
                      ┌─────────────────┐
                      │ cancel_queue     │
                      └────────┬────────┘
                               │
                               ↓
                      ┌────────────────────┐
                      │ 订单取消消费者      │
                      │ → 检查支付状态      │
                      │ → 已支付：忽略      │
                      │ → 未支付：取消 + 还 │
                      │   库存             │
                      └────────────────────┘
```

```java
// 延时队列配置代码
// 1. 声明实际的"死信"交换机（用于接收过期消息）
channel.exchangeDeclare("order_actual_exchange", "direct", true);

// 2. 声明延时队列（核心！x-message-ttl + x-dead-letter-exchange）
Map<String, Object> args = new HashMap<>();
args.put("x-message-ttl", 30 * 60 * 1000);        // 30分钟 TTL
args.put("x-dead-letter-exchange", "order_actual_exchange"); // 过期后去哪个交换机
args.put("x-dead-letter-routing-key", "order.cancel");      // 过期后的 routing key
channel.queueDeclare("order_delay_queue", true, false, false, args);

// 3. 绑定延时队列
channel.queueBind("order_delay_queue", "delay_exchange", "order.delay");

// 4. 业务消费者监听实际的取消队列
channel.queueDeclare("order_cancel_queue", true, false, false, null);
channel.queueBind("order_cancel_queue", "order_actual_exchange", "order.cancel");
```

**延时队列的局限与方案对比：**

| 方案 | 精度 | 灵活性 | 复杂度 | 适用场景 |
|------|------|--------|--------|----------|
| **RabbitMQ TTL+DLX** | 秒级 | ❌（所有消息同一 TTL） | 低 | 固定延迟（30 分钟取消订单） |
| **RabbitMQ 插件** `rabbitmq_delayed_message_exchange` | 秒级 | ✅（每条消息不同延迟） | 低 | 不同业务有不同延迟 |
| **RocketMQ 延迟消息** | 18 级固定延迟 | 中（1s~2h 共 18 级） | 低 | 大多数延迟场景够用 |
| **Redis ZSet + 定时轮询** | 毫秒级 | ✅ 极高 | **高** | 需要毫秒精度的延迟任务 |

> 🔑 **面试陷阱：** "RabbitMQ 延时队列能不能每条消息设置不同延迟？"
> → 普通 TTL+DLX 方案**不行**，所有消息同一 TTL。要实现不同延迟，要么用 `rabbitmq_delayed_message_exchange` 插件，要么多建几个不同 TTL 的队列，要么直接用 RocketMQ。

---

## 六、面试真题与回答框架

### 🔴 Q1：RabbitMQ 如何保证消息不丢失？（全链路）

> 频率：⭐⭐⭐⭐⭐ | 字节/阿里/美团必问

**标准回答框架（按消息流转三阶段）：**

> "消息丢失分三个阶段——**生产、存储、消费**。我们分别做了保障：
>
> **生产端：** 开启 Publisher Confirm 异步模式。Producer 发消息后，Broker 返回 ACK 才算成功；收到 NACK 或超时则重发。
>
> **存储端/Broker 端：** 两个持久化——队列 durable=true，消息 deliveryMode=2。加上镜像队列或 Quorum Queue，多节点 Raft 复制，过半节点确认才提交。
>
> **消费端：** 关闭自动 ACK，手动确认。业务处理成功后调 basicAck，失败调 basicNack 重新入队。配合 prefetch=1 防止消费者 OOM，并保证消息不丢。另外消费逻辑必须做**幂等**——数据库唯一约束或 Redis SETNX 防重。"

---

### 🔴 Q2：Exchange 有几种类型？分别什么场景？

> 频率：⭐⭐⭐⭐⭐

| 类型 | 规则 | 场景 |
|------|------|------|
| **Direct** | Routing Key 精确匹配 | 单播任务分发 |
| **Topic** | 模式匹配：`*` 一级、`#` 多级 | 多级分类路由（日志系统） |
| **Fanout** | 忽略 Key，广播到所有绑定队列 | 配置热更新、通知广播 |
| **Headers** | Header 属性匹配（几乎不用） | 了解即可 |

> 加分：说一个你项目里具体用的场景——比如"订单系统用 Topic Exchange，routing key 设计为 `order.{操作}.{状态}`"

---

### 🔴 Q3：RabbitMQ 的延时队列怎么实现？和 RocketMQ 对比？

> 频率：⭐⭐⭐⭐ | 场景设计题

**回答要点：**
1. 先说方案：TTL + DLX（死信交换机），消息设置过期时间，过期后自动转入死信队列
2. 说局限：同一队列只能一个 TTL，不同延迟需要不同队列或插件
3. 说对比：RocketMQ 原生支持 18 级延迟消息，开箱即用

---

### 🔴 Q4：prefetch=1 有什么作用？

> 频率：⭐⭐⭐ | 容易冷门

> "prefetch=1 是 Consumer 端的 QoS 限流——告诉 Broker 一次只发一条消息给这个消费者，只有收到 ACK 后才发下一条。作用是**自动负载均衡**：处理快的消费者拿得多，慢的拿得少，实现'能者多劳'。同时也能防止消费者 OOM。"

---

### 🟡 Q5：Connection 和 Channel 的区别？为什么一个 Connection 可以有多个 Channel？

> 频率：⭐⭐⭐

- **Connection**：TCP 长连接，创建成本高
- **Channel**：虚拟连接，复用同一 TCP，创建成本低
- **为什么？** 类似 HTTP/2 多路复用——减少握手开销，一个应用进程建立一个 Connection 就够了
- **注意**：Channel 非线程安全，多线程要各用各的 Channel

---

### 🟡 Q6：autoACK 和手动 ACK 的区别？什么时候用哪种？

| | autoACK | 手动 ACK |
|---|---------|----------|
| **删除时机** | 消息**发到消费者**即删除 | 消费者**调 basicAck 后**删除 |
| **风险** | 消费者 OOM / 处理失败消息已丢 | 忘记 ACK 导致消息堆积 |
| **适用** | 允许丢失的场景（如非关键日志） | **业务数据必用手动 ACK** ✅ |

---

### 🟡 Q7：什么是死信队列？消息什么时候变死信？

> 频率：⭐⭐⭐

三种情况：被拒绝（requeue=false）、消息 TTL 过期、队列满了。

用途：延时队列、异常消息隔离、监控告警。

---

### 🟢 Q8：如何保证消息不被重复消费？（幂等性）

> 频率：⭐⭐⭐⭐ | 详细方案见 [[03-消息队列通用问题]]

RabbitMQ 层面方案：数据库唯一约束（msg_id 唯一键）、Redis SETNX 记录已消费消息 ID、业务状态机（状态已变则跳过）。

---

## 面试话术：如何保证消息不丢失

> "对于消息可靠性，我们从**生产、存储、消费**三个阶段做了全链路保障。
>
> **生产端：** 开启了 Publisher Confirm 机制，消息发送后异步等待 Broker 确认回执。
>
> **存储端：** 队列和消息都设置了持久化（durable=true, deliveryMode=2）。
>
> **消费端：** 关闭自动 ACK，改为手动确认。业务处理成功后才 basicAck。
>
> 这样三层防护下来，消息丢失概率降到非常低。"

---

## 面试前 5 分钟速览

**架构层：**
- [ ] Connection（TCP 长连接）vs Channel（虚拟连接，非线程安全，一个线程一个）
- [ ] vhost = 迷你 RabbitMQ，独立 Exchange/Queue/权限

**路由层（四种 Exchange）：**
- [ ] Direct = 精确匹配、Topic = 模式 `*`/`#`、Fanout = 广播、Headers = 几乎不用
- [ ] **陷阱：** Fanout 的 Routing Key 传了也白传

**可靠性层（防丢三阶段）：**
- [ ] 生产端 → **Publisher Confirm**（异步模式最佳）
- [ ] 存储端 → **持久化**（队列 durable + 消息 deliveryMode=2）+ **Quorum Queue**
- [ ] 消费端 → **手动 ACK** + **QoS prefetch** + **幂等**（DB唯一键/Redis SETNX）
- [ ] Confirm 三种模式：单条同步(慢) < 批量同步(中) < 异步(快，推荐)

**死信 & 延时：**
- [ ] 死信来源：被拒(requeue=false) / TTL过期 / 队列满
- [ ] 延时队列 = TTL + DLX，**局限**：同队列只能一个 TTL
- [ ] 不同 MQ 延时对比：RabbitMQ(插件) / RocketMQ(18级原生) / Redis ZSet(最灵活)

---

## 记忆口诀

```
四种交换记分明，直接主题扇出头
Confirm 持久加手动，三关守住不丢丢
死信三源延时代，TTL 加 DLX 够
prefetch 一是负载，能者多劳自动有
Quorum 替代镜像队，Raft 协议更可靠
```

---

> 📖 上一篇：[[01-消息队列概述]] | 下一篇：[[03-消息队列通用问题]]
