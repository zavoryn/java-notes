# 颐享/知光 (YiXiang/ZhiGuang) - 知识分享社区

## 项目代码位置
- 后端：`E:\code\yixiang\yixiang_be`
- 前端：`E:\code\yixiang\yixiang_fe`

## 项目概述
面向技术人群（Java、AI、Agent）的知识分享社区平台。用户可以发布"知文"（图文帖子），进行点赞、收藏、关注等社交互动，并支持 AI 驱动的 RAG 问答。

## 技术栈

### 后端（Java 21 + Spring Boot 3.2.4）
| 类别 | 技术 |
|------|------|
| 框架 | Spring Boot 3.2.4, Spring Security |
| ORM | MyBatis 3.0.3 |
| 数据库 | MySQL 9.5 |
| 缓存 | Caffeine (L1) + Redis (L2) |
| 分布式锁 | Redisson 3.52 |
| 消息队列 | Spring Kafka |
| 搜索 | Elasticsearch 9.2 |
| AI | Spring AI 1.0.3 + DeepSeek |
| 对象存储 | 阿里云 OSS |
| Binlog CDC | 阿里 Canal 1.1.8 |
| 认证 | JWT RS256 双 Token |

### 前端（React 18 + TypeScript）
| 类别 | 技术 |
|------|------|
| 框架 | React 18 + TypeScript |
| 构建 | Vite 5 |
| 路由 | react-router-dom v6 |
| 样式 | CSS Modules |
| Markdown | react-markdown + remark-gfm |
| 状态管理 | React Context + useState |

## 数据库表（5张）
1. **users** - 用户表（phone/email 双通道注册）
2. **login_logs** - 登录审计日志
3. **know_posts** - 知文内容（雪花ID, OSS存储, 草稿/发布状态）
4. **outbox** - 事务发件箱（保证最终一致性）
5. **following/follower** - 关注关系（双表冗余，读优化）

## 核心模块

### 1. 认证模块 (auth/)
- JWT RS256 双 Token（access 15min + refresh 7day）
- Refresh Token Redis 白名单
- 邮箱验证码注册/登录（Redis 存储，限频限次）
- 密码策略：8位+字母数字混合，bcrypt strength 12

### 2. 知文模块 (knowpost/)
- 完整发布流程：创建草稿 → 预签名上传 → 确认内容 → 设置元数据 → 发布
- Feed 流：三级缓存（Caffeine → Redis 片段缓存 → DB）
- 热点检测：滑动窗口计数，动态延长缓存 TTL
- 单飞锁：ConcurrentHashMap 分页互斥，防缓存击穿

### 3. 计数模块 (counter/)
- Bitmap 分片幂等判重（32K bit/片，用户ID取模分片）
- SDS 固定结构存储（20字节打包5个计数器）
- Kafka 异步聚合（toggle → Kafka → Hash 聚合桶）
- 重建机制：Redisson 分布式锁 + 令牌桶限流 + 指数退避

### 4. 关系模块 (relation/)
- 关注/取关：令牌桶限流（容量100, 1/s）
- 双表冗余写（following + follower）
- 事务发件箱 + Canal CDC → Kafka → 消费者更新缓存
- Redis ZSet 缓存（score 为时间戳），DB 回源回填

### 5. 搜索模块 (search/)
- Elasticsearch 全文检索（multi_match: title^3 + body）
- function_score 排序：log1p(like_count) + log1p(view_count)
- completion suggester 自动补全
- search_after 游标分页

### 6. AI/RAG 模块 (llm/)
- Markdown 分块：按标题边界 + 固定长度（800字, 100重叠）
- 向量索引：DeepSeek Embedding → ES Vector Store
- 指纹去重：SHA-256/ETag 判定是否需要重新索引
- RAG 问答：SSE 流式输出，限制仅回答检索到的上下文

### 7. 存储模块 (storage/)
- 阿里云 OSS 预签名 URL 直传
- 客户端直传 bypass 应用服务器
- 确认时校验 ETag、size、SHA-256

## API 路由总览
| 路径前缀 | 功能 |
|---------|------|
| `/api/v1/auth` | 认证（注册/登录/登出/刷新Token） |
| `/api/v1/knowposts` | 知文 CRUD + Feed + 发布流程 |
| `/api/v1/action` | 点赞/取消点赞/收藏/取消收藏 |
| `/api/v1/counter` | 计数查询 |
| `/api/v1/profile` | 用户资料编辑 |
| `/api/v1/relation` | 关注/取关/关注列表 |
| `/api/v1/search` | 全文搜索 + 自动补全 |
| `/api/v1/storage` | 预签名上传 |

## 前端页面
| 路由 | 页面 |
|------|------|
| `/` | 首页 Feed 流 |
| `/search` | 全文搜索 |
| `/create` | 发布知文 |
| `/learn` | 学习路径 + 收藏 |
| `/profile` | 个人主页 |
| `/post/:id` | 知文详情 + RAG 问答 |

## 关联知识点
- > 关联：[juc并发编程](../../juc并发编程/) — Redis 单飞锁、ConcurrentHashMap 分页互斥
- > 关联：[jvm虚拟机](../../jvm虚拟机/) — Caffeine 本地缓存、SDS 固定结构内存布局
- > 关联：[mysql](../../mysql/) — 事务发件箱模式、双表冗余设计、MyBatis XML 映射
- > 关联：[redis](../../redis/) — 三级缓存、Bitmap 分片、ZSet 缓存、令牌桶限流、Redisson 分布式锁
- > 关联：[消息队列](../../消息队列/) — Kafka 异步聚合、Canal CDC、消费者幂等
- > 关联：[计算机网络](../../计算机网络/) — SSE 流式推送、预签名 URL 直传、JWT RS256
- > 关联：[spring](../../spring/) — Spring Security 无状态认证、Spring AI、Spring Kafka
- > 关联：[设计模式](../../设计模式/) — 发件箱模式、策略模式（验证码发送）、观察者模式（事件驱动计数）
