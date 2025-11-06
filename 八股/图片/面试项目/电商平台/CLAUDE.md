# 电商数据分析系统 (ecom-analytics)

## 项目代码位置
- 后端：`E:\code\dianshang`
- GitHub：*(待补充)*

## 项目概述
电商数据分析后端系统，覆盖用户行为采集、大数据清洗存储、多维度聚合查询。微服务架构，7个模块协同工作。源自真实实习经历，专为面试设计，每个核心类都标注了对应面试稿章节。

## 技术栈
| 类别 | 技术 | 版本 |
|------|------|------|
| 语言 | Java | 17 |
| 框架 | Spring Boot | 3.2.5 |
| 微服务 | Spring Cloud + Spring Cloud Alibaba | 2023.0.1 |
| 注册中心 | Nacos | 2.3 |
| 网关 | Spring Cloud Gateway (WebFlux) | -- |
| 限流 | Sentinel | 1.8.6 |
| ORM | MyBatis-Plus 3.5.5 + JdbcTemplate 双数据源 | -- |
| 关系库 | MySQL | 8.0 |
| OLAP | ClickHouse | 24.x |
| 搜索 | Elasticsearch | 8.13 |
| 缓存 | Redis | 7 |
| 业务 MQ | RocketMQ | 5.1.4 |
| 大数据 MQ | Kafka | 3.6 |
| 流计算 | Apache Flink | 1.18.1 |
| API 文档 | Knife4j (OpenAPI 3) | 4.5.0 |

## 模块架构（7个模块）
| 模块 | 端口 | 职责 |
|------|------|------|
| `ecom-analytics-common` | -- | 共享 DTO、枚举、统一响应、工具类 |
| `ecom-analytics-gateway` | 8080 | API 网关：路由、Sentinel 限流、访问日志 |
| `ecom-analytics-collector` | 8081 | 数据采集：埋点事件接收、订单同步、本地日志兜底、MQ 发送 |
| `ecom-analytics-processor` | 8082 | 数据处理：MQ 消费、MySQL+ClickHouse 双写、定时聚合/补偿 |
| `ecom-analytics-query` | 8083 | 查询 API：趋势/漏斗/运营看板/排行榜 + Redis 缓存 |
| `ecom-analytics-search` | 8084 | ES 搜索：商品全文检索、订单 search_after 深分页 |
| `ecom-analytics-bigdata` | -- | 大数据层骨架：Kafka 生产者、Flink 清洗作业、Canal 同步 |

## 存储设计

### MySQL（11张表）
1. **event_detail_YYYYMM** - 用户行为明细（按月分表，JSON ext_info，request_id 幂等唯一键）
2. **event_agg_daily** - 日粒度商品聚合（pv/uv/搜索/加购/支付/gmv）
3. **order_sync** - 订单数据（version 乐观锁 + ON DUPLICATE KEY UPDATE）
4. **join_temp_event** - 双流 Join 临时表（0等待→1匹配/2重试/3死信）
5. **id_mapping** - 设备→用户 ID 映射（OneID）
6. **product_info** - 商品目录
7. **category_daily** - 品类日聚合
8. **search_keyword_daily** - 搜索热词日聚合（搜索量/uv/点击/ctr/cvr）
9. **item_ranking_daily** - 商品日排行 TOP100（gmv/支付/浏览/加购多维度）
10. **platform_daily** - 平台级日指标（pv/uv/gmv/支付率/arpu/退款）
11. **data_verify_report** - 数据对账报告

### ClickHouse（6张表）
1. **events_local** - 核心事件明细（MergeTree，按月分区，TTL 180天）
2. **funnel_agg_state** - 漏斗聚合物化视图（AggregatingMergeTree）
3. **dwd_user_order_action** - DWD 层宽表（Flink 双流 Join 结果）
4. **dws_item_daily** - DWS 层商品日汇总（SummingMergeTree）
5. **dws_category_daily** - DWS 层品类日汇总
6. **dws_search_keyword_daily** - DWS 层热词日汇总

### Elasticsearch（2个索引）
1. **product_index** - 商品搜索（IK 分词：ik_max_word 索引 + ik_smart 搜索）
2. **order_event_index** - 订单事件搜索（search_after 深分页）

## 核心技术亮点

### 1. 三层幂等保障
Redis SETNX（5min TTL）→ DB UNIQUE KEY → REPLACE INTO。Redis 故障自动降级为 DB 兜底。

### 2. 双流 Join + 补偿机制
用户行为和订单数据分别从两条 MQ 到达。临时表存储半匹配记录，每小时扫描任务补偿未匹配记录，重试上限 + 死信标记。

### 3. 三级查询策略
聚合表（预计算，<100ms）→ 实时明细表 → Redis 缓存。用于趋势查询和排行榜。

### 4. 按月分表 + 跨月 ES 委托
`event_detail_YYYYMM` 按月路由。跨月查询委托 ES search_after，避免 UNION ALL + 内存合并 OOM。

### 5. MySQL + ClickHouse 双写
`EventPersistService` 同时写入 MySQL（事务明细）和 ClickHouse（OLAP 分析）。

### 6. 订单版本乐观锁
`INSERT ... ON DUPLICATE KEY UPDATE` + version 字段，`IF(VALUES(version) > version, ...)` 防乱序更新。

### 7. 本地缓冲兜底
`LocalBufferFallback` 内存队列（10K 容量）兜底 MQ 发送失败，1秒批量重试。

### 8. 网关按路由限流
Redis 令牌桶：collector 1000/s、query 200/s、search 300/s。

### 9. 多级缓存 TTL
7 个缓存层级，TTL 从 5min（热词）到 60min（品类统计）。

### 10. 五步日聚合流水线
`DailyAggregateTask` 每日 00:30 执行：商品日 → 品类日 → 热词 → 排行榜 → 平台日，含重试逻辑。

### 11. search_after 深分页
ES composite sort key（如 `[sales30d DESC, itemId ASC]`）实现恒定性能深分页，避免 LIMIT OFFSET 问题。

### 12. 订单双保险
RocketMQ 主动推送 + `OrderPullTask` 每 5 分钟被动拉取。游标存 Redis + MySQL 回源恢复。

## API 路由
| 路径 | 功能 |
|------|------|
| `POST /api/collect/event` | 单条埋点事件采集 |
| `POST /api/collect/event/batch` | 批量事件采集 |
| `POST /api/sync/order` | 订单同步 |
| `POST /api/collect/login` | 设备-用户绑定（OneID） |
| `GET /api/query/item-trend` | 商品趋势 |
| `GET /api/query/funnel` | 漏斗分析（ClickHouse windowFunnel） |
| `GET /api/operation/overview` | 运营概览（GMV/支付率/ARPU） |
| `GET /api/operation/gmv-trend` | GMV 趋势 |
| `GET /api/ranking/top-items` | 商品排行 TOP100 |
| `GET /api/ranking/hot-keywords` | 热词排行 |
| `GET /api/search/product` | 商品 ES 搜索 |
| `GET /api/search/order` | 订单 ES 搜索 |

## 关联知识点
- > 关联：[微服务](../../微服务/) — Spring Cloud Gateway 路由、Nacos 注册发现、Sentinel 限流熔断
- > 关联：[mysql](../../mysql/) — 按月分表、乐观锁、ON DUPLICATE KEY UPDATE、HikariCP 双数据源
- > 关联：[redis](../../redis/) — 三层幂等 SETNX、7级缓存 TTL、令牌桶限流、游标存储
- > 关联：[消息队列](../../消息队列/) — RocketMQ 顺序消费/重试、Kafka 高吞吐 event streaming、本地缓冲兜底
- > 关联：[计算机网络](../../计算机网络/) — WebFlux 网关、SSE、RESTful API 设计
- > 关联：[juc并发编程](../../juc并发编程/) — 内存队列、批量刷写、双写一致性
- > 关联：[spring](../../spring/) — Spring Boot 3.2、@ConfigurationProperties、Spring Cache、定时任务
- > 关联：[设计模式](../../设计模式/) — 策略模式（存储路由）、模板方法（聚合流水线）、工厂模式
- > 关联：[jvm虚拟机](../../jvm虚拟机/) — 跨月查询内存控制、大批量聚合内存优化
