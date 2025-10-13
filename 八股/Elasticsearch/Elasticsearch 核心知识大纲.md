---
tags:
  - 八股
  - elasticsearch
  - 数据库
created: 2026-04-28
---

# Elasticsearch 核心知识大纲

## 一、基础概念（先理解 ES 是什么）

### 1.1 什么是 Elasticsearch
- 基于 **Lucene** 的分布式**搜索引擎**，本质是一个 **NoSQL 文档数据库**
- 核心能力：近实时（NRT, Near Real Time）的全文搜索、聚合分析
- 技术栈：ELK = **E**lasticsearch + **L**ogstash + **K**ibana
- RESTful API，JSON 交互，Java 开发

### 1.2 核心概念对比（用 MySQL 类比，快速入门）
| Elasticsearch | MySQL | 说明 |
|:---|:---|:---|
| Index（索引） | Database | 一类文档的集合 |
| Type（已废弃） | Table | 7.x 以后一个 Index 只有一个 Type（_doc） |
| Document（文档） | Row | 一条数据，JSON 格式 |
| Field（字段） | Column | 文档中的属性 |
| Mapping（映射） | Schema | 字段类型定义 |
| DSL（查询语言）| SQL | 查询语法 |
| Shard（分片） | 分库 | 水平拆分 |

### 1.3 ES 和 MySQL 的根本区别
- **MySQL**：正向索引，按 ID 找数据快 → `select * from table where id = 1`
- **ES**：倒排索引，按内容找数据快 → `搜索"八股"找出所有包含它的文档`

> 💡 **一条数据你很少用"ID"去搜，更多是用"关键词"搜 → 这就是 ES 存在的意义**

---

## 二、倒排索引（最核心底层原理）

### 2.1 什么是倒排索引
正常（正排）索引：`文档ID → 内容`
倒排索引：**`词条(Term) → 文档ID列表`**

### 2.2 构建过程（三个步骤）
1. **分词（Tokenization）**：把文档内容拆成一个个词条（Term）
2. **归一化（Normalization）**：大小写转换、去除停用词、词干提取
3. **建立倒排表**：
   ```
   词条"java"    → [文档1, 文档3, 文档5]   位置: [2, 5, 1]
   词条"八股"    → [文档1, 文档2]          位置: [4, 3]
   词条"redis"   → [文档2, 文档4, 文档5]   位置: [1, 6, 3]
   ```

### 2.3 倒排索引内部结构（面试重点）
```
Term Dictionary（词项字典）
    ↓ (二分查找，存在内存)
Term Index（词项索引，FST 压缩前缀树）
    ↓
Posting List（倒排列表=文档ID+词频+位置+偏移量）
    ↓ (压缩：FOR编码 + Roaring Bitmap)
磁盘上的 .doc/.pos/.pay 文件
```

### 2.4 为什么 ES 搜索快？
- **倒排索引**：直接查找"词→文档"映射，不需要全表扫描
- **FST**：高效压缩前缀树，快速定位 Term
- **跳表（Skip List）**：在 Posting List 里加速合并
- **位图压缩**：Roaring Bitmap 节省内存，加速交集/并集

---

## 三、集群架构

### 3.1 节点类型
| 节点类型 | 职责 |
|:---|:---|
| **Master Node** | 管理集群元数据（索引创建/删除、分片分配），不处理数据 |
| **Data Node** | 存储数据、执行 CRUD、搜索聚合 |
| **Ingest Node** | 数据预处理（管道 pipeline） |
| **Coordinating Node** | 请求路由、结果聚合（每个节点默认都是） |

### 3.2 分片（Shard）机制
- **主分片（Primary Shard）**：实际存储数据的容器
  - 索引创建时确定，**不可修改**（所以建索引时要规划好）
  - 默认 1 个，建议按数据量估算
- **副本分片（Replica Shard）**：主分片的拷贝
  - 提供**高可用**（主分片挂了副本接替）
  - 提供**读扩展**（搜索可以从副本读）
  - **可动态调整**

### 3.3 分片路由
```
shard = hash(_routing) % number_of_primary_shards
```
- 默认 _routing 就是文档 _id
- ⚠️ 这也是为什么主分片数不能改——hash 取模的分母变了，路由全乱

### 3.4 集群健康状态
| 颜色 | 含义 |
|:---|:---|
| Green | 所有主/副本分片都正常 |
| Yellow | 主分片正常，部分副本未分配（单节点必然 yellow） |
| Red | 有主分片丢失，数据不完整 |

---

## 四、写入流程（文档如何存进 ES）

```
Client 发写入请求
        ↓
① 协调节点：hash(_routing) 算出目标分片 → 转发请求
        ↓
② 主分片：写 translog（WAL预写日志）→ 写内存 buffer
        ↓
③ 主分片返回成功 → 协调节点 → Client 收到 ack
        ↓ (并行)
④ 副本分片：同步写入
        ↓ (后台异步)
⑤ refresh（每1s）：buffer → segment（内存中，可搜索）    ← 这就是"近实时(NRT)"
        ↓ (后台异步)
⑥ flush（每5s或translog满）：translog → 磁盘，清空 translog
```

### 4.1 关键时间点（面试问"数据多久能搜到"）
| 操作 | 默认间隔 | 作用 |
|:---|:---|:---|
| **refresh** | 1 秒 | 数据从 buffer 刷到 segment，**变得可搜索** |
| **flush** | translog 达到 512MB 或 5 秒 | 持久化到磁盘 |
| **translog 同步** | 每个请求（index 操作） | 保证不丢数据 |

### 4.2 写入性能优化
- 批量写入（Bulk API）
- 减少 refresh 频率（index.refresh_interval = 30s）
- 副本设为 0 写完再加（`number_of_replicas=0`）
- 使用 SSD

### 4.3 数据可靠性（不丢数据）
- **translog（事务日志）**：类比 MySQL 的 binlog，写入前先写 translog
- 每次写操作都 fsync 到磁盘（`index.translog.durability = request`）
- 宕机恢复时重放 translog

---

## 五、查询流程（搜索怎么跑的）

### 5.1 Query（查询）阶段
```
协调节点 → 广播到所有相关分片（可以是主或副本，轮询负载均衡）
    → 每个分片本地搜索，返回 (doc_id + _score) 的排序列表
    → 协调节点合并排序，选出 top N
```

### 5.2 Fetch（取数据）阶段
```
协调节点 → 根据 doc_id 去对应分片获取完整文档
    → 返回给客户端
```

### 5.3 相关性打分（TF-IDF / BM25）
- ES 5.0 以前用 TF-IDF，之后默认 **BM25**
- 核心思想：**词在文档中出现越频繁（TF）**，文档越相关；但**词在所有文档中越常见（IDF）**，权重越低
- BM25 引入长度惩罚，短文档匹配得分更高

### 5.4 搜索类型
| 类型 | 说明 |
|:---|:---|
| **query** | 搜索结果，重相关性（_score 排序） |
| **filter** | 精确匹配，不计算分数，有缓存（更快） |

---

## 六、数据类型与 Mapping

### 6.1 核心数据类型
| 类型 | 场景 |
|:---|:---|
| **text** | 全文搜索（会分词） |
| **keyword** | 精确匹配（聚合、排序、过滤，不分词） |
| **long / integer / float / double** | 数值 |
| **date** | 日期时间 |
| **boolean** | 布尔 |
| **geo_point** | 地理位置 |

### 6.2 text vs keyword（面试高频）
```
"中华人民共和国"
   ↓ text 类型（分词）
["中华", "人民", "共和国"]  → 搜"人民"能命中
   ↓ keyword 类型（不分词）
"中华人民共和国"             → 搜"人民"命中不了，必须搜完整字符串
```

### 6.3 Mapping 的创建和坑
- **动态映射（Dynamic Mapping）**：自动推断类型，但可能猜错会引发线上故障
- **显式映射（Explicit Mapping）**：手动定义，生产环境必须这么做
- ⚠️ **Mapping 无法修改**：只能新建索引 + reindex 迁移数据

---

## 七、分词器（Analyzer）

### 7.1 分词器组成
```
Analyzer = Character Filter + Tokenizer + Token Filter
           （字符过滤）   （分词器）    （词条过滤）
```

### 7.2 常用分词器
| 分词器 | 特点 |
|:---|:---|
| **standard** | 默认，英文按空格分词 |
| **ik_max_word** | 中文细粒度分词（最广匹配） |
| **ik_smart** | 中文粗粒度分词 |
| **pinyin** | 拼音分词 |

### 7.3 分词器使用场景
- **ik_max_word** + **pinyin** → 搜索场景（尽量多匹配）
- **keyword** → 聚合、排序、精确匹配

---

## 八、聚合分析（Aggregation）

### 8.1 三种聚合
| 类型 | 说明 | 举例 |
|:---|:---|:---|
| **Bucket** | 分桶（类似 GROUP BY） | 按城市分组 |
| **Metric** | 指标计算 | COUNT, AVG, MAX |
| **Pipeline** | 管道（对聚合结果再聚合） | 月环比 |

### 8.2 聚合注意事项
- 聚合对内存消耗大
- 尽量用 **doc_values**（列存，默认开启（keyword/number/date））
- 避免在高基数（cardinality）字段上做 terms 聚合

---

## 九、性能优化（面试重点）

### 9.1 写入优化
- 批量写入（Bulk）
- 增大 refresh_interval
- 减少副本数（写完再加回去）
- 调大 translog flush 阈值
- 使用自动生成的 ID（比指定 ID 快）

### 9.2 查询优化
- 能用 **filter** 就用 filter（不用算分 + 有缓存）
- 减少返回字段（`_source: ["field1"]`）
- 分页场景用 **search_after** 替代深度分页（from + size）
- 避免 `*` 开头的前缀查询（`prefix`）
- 合理设置分片数（分片多→并发高，但合并开销也大）
- 冷热分离架构

### 9.3 深度分页问题（面试常问）
```
问题：from + size 取第 10000 条数据，为什么会变慢？
答案：ES 需要在每个分片上取 from + size 条（比如5个分片就是50000条），
     再到协调节点排序取前 10010 条，内存和网络开销巨大

解决方案：
  - search_after：基于上一页的排序值继续翻（走游标，实时）
  - scroll：快照查询（适合全量导出，非实时）
```

### 9.4 分片数规划
```
单分片建议不超过 3️⃣0️⃣ GB（Lucene 段合并性能下降）
分片数 ≈ 数据总量 / 30GB 向上取整，再适当+2预留
```

---

## 十、ES vs MySQL 使用场景

| 场景 | 选型 | 原因 |
|:---|:---|:---|
| 商品搜索、文章搜索 | **ES** | 全文检索、分词、高亮 |
| 订单管理后台 | **MySQL** | 事务、精确查询 |
| 日志分析 | **ES (+ Logstash + Kibana)** | 海量数据、时序、聚合 |
| 用户认证 | **MySQL** | 事务、精确、一致性强 |
| 搜索推荐 | **ES** | 倒排索引 + 相关性打分 |

> 🎯 经典架构 MySQL 做主存储，ES 做搜索索引，通过 **Canal / MQ 同步数据**

---

## 十一、面试高频面试题（目标：能说清楚）

1. **倒排索引原理**：没回答到这个就是不及格
2. **写入流程**：translog → buffer → refresh → flush，每个阶段干了什么
3. **为什么 ES 是近实时（NRT）不是实时**：因为默认 1s refresh
4. **ES 如何保证数据不丢**：translog 机制
5. **深度分页问题及解决**：from/size 开销 + search_after/scroll
6. **text vs keyword 区别**：分词 vs 不分词
7. **分片为什么不能改**：hash(_routing) % shard_num
8. **ES 集群高可用**：主分片 + 副本分片，master 选举
9. **MySQL 数据如何同步到 ES**：Canal（binlog）+ MQ / 双写 + MQ
10. **BM25 是什么**：TF-IDF 的改进版，引入长度惩罚，默认算法

---

## 📚 学习路线建议

```
第1步：理解核心概念（倒排索引 > 索引/文档/映射）
第2步：搞清写入流程（translog → buffer → refresh → flush）
第3步：搞清查询流程（Query → Fetch 两阶段）
第4步：上手搭建集群（单节点 → 三节点）
第5步：实践优化（分片规划、查询优化）
第6步：MySQL 同步 ES 实战
```

> 🔗 配合阅读：[[Elasticsearch 面试题汇总]]
