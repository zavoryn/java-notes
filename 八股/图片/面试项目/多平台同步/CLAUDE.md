# 多平台同步 (Multi-Shop Sync) - 多电商渠道商品同步

## 项目代码位置
- 后端：`E:\agent\多平台同步`
- GitHub：`https://github.com/CHEN666333-SVG/multi-shop-sync.git`

## 项目概述
统一的多电商渠道商品同步框架，支持将本地商品数据同步到抖音小店、小红书、微信小店等多个平台。管理商品全生命周期：创建/推送、上下架状态变更、平台审核回调、状态双向同步。

## 技术栈
| 类别 | 技术 |
|------|------|
| 语言 | Java 8 |
| 框架 | Spring Boot 2.7.18 |
| HTTP | Spring WebFlux WebClient |
| JSON | FastJSON2 2.0.43 |
| 工具 | Hutool 5.8.25 |
| 重试 | Spring Retry + AOP |
| 数据库 | MySQL (InnoDB, utf8mb4)，Schema 已设计但 ORM 层未接入 |

## 数据库设计（4张表）
1. **channel_product_mapping** - 本地商品与平台商品映射，双状态追踪（local_status + external_status）
2. **channel_sku_mapping** - 本地 SKU 与平台 SKU 映射，价格/库存追踪
3. **category_mapping** - 本地分类与平台分类映射
4. **callback_log** - 平台回调日志审计（raw_data JSON + 处理状态）

## 核心架构

### 设计模式组合
- **策略模式** (`IPlatformProductStrategy`) — 每个平台一套策略实现（pushProduct/changeStatus/syncPlatformStatus/parseCallback）
- **模板方法** (`AbstractPlatformStrategy`) — 统一参数校验、结构化日志、`@Retryable`（3次重试，1s退避）、异常包装
- **工厂模式** (`PlatformStrategyFactory`) — Spring ApplicationContextAware 自动发现 @Component 策略 Bean，Map<O(1) 查找
- **防腐层 (ACL)** — 每个策略将内部 `StandardProductDTO` 翻译为平台特定 API 格式

### 支持渠道
| 渠道 | 认证方式 | 推送API | 回调事件 |
|------|---------|---------|---------|
| 抖音小店 | HMAC-SHA256 签名 | `product.addV2` | 审核通过/拒绝/上下架 |
| 小红书 | MD5 签名 | `product.createItemAndSku` | 买able/审核拒绝/创建 |
| 微信小店 | Access Token (`client_credential`) | `/channels/ec/product/add` | 审核通过/拒绝/下架 |
| 本地商城 | 无（直连DB） | 本地持久化 | 无 |

### 商品状态机
```
DRAFT(0) → pushProduct → WAIT_AUDIT(10) → 审核通过 → ON_SHELF(20)
                                          → 审核拒绝 → AUDIT_REJECT(30)
ON_SHELF(20) → changeStatus → OFF_SHELF(40)
OFF_SHELF(40) → 重新上架 → ON_SHELF(20)
```

## API 路由
| 路径 | 功能 |
|------|------|
| `POST /api/product/push` | 推送商品到单个渠道 |
| `POST /api/product/list` | 批量推送到多渠道 |
| `POST /api/product/changeStatus` | 统一上下架控制 |
| `POST /api/product/syncStatus` | 主动同步平台状态（补偿） |
| `POST /api/webhook/{channel}` | 通用平台回调（异步处理） |
| `POST /api/webhook/wechat` | 微信专用回调（XML响应） |

## 实现状态
- **已完成**：策略/模板/工厂框架、4个渠道策略实现、REST 控制器、Webhook 异步处理、数据库 Schema、三平台认证（HMAC-SHA256/MD5/AccessToken）
- **未完成（TODO）**：数据库持久层（无 ORM 依赖）、微信回调 AES 解密验证、状态解析、分类映射查询、Token 自动刷新、限流/Sentinel、MQ 延迟重试、XXL-JOB 定时补偿

## 关键设计决策
1. **价格单位统一为"分"** — 避免元/分转换 bug
2. **`extraAttrs` Map** — `StandardProductDTO` 上的扩展字段，透传平台特有属性
3. **Webhook 异步处理** — 立即返回 200，专用线程池处理（4核心/8最大/队列100），满足各平台 SLA（微信5s、小红书2s）
4. **Spring 自动发现注册** — 新增渠道只需创建 @Component 类，工厂零改动

## 关联知识点
- > 关联：[设计模式](../../设计模式/) — 策略模式 + 模板方法 + 工厂模式三件套组合
- > 关联：[spring](../../spring/) — Spring Boot 2.7、@Retryable、WebClient、@ConfigurationProperties
- > 关联：[mysql](../../mysql/) — 渠道映射表设计（双状态追踪）、回调日志审计
- > 关联：[计算机网络](../../计算机网络/) — HMAC-SHA256/MD5 签名认证、Webhook 回调、Access Token 管理
- > 关联：[消息队列](../../消息队列/) — TODO: MQ 延迟重试方案
- > 关联：[微服务](../../微服务/) — 防腐层设计、多渠道适配
