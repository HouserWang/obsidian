# Elasticsearch 核心架构与原理

## 📌 第一性原理

**基于 Lucene 的分布式搜索引擎**：通过倒排索引实现快速全文检索，通过[[Sharding & Partitioning (分片：分区)|分片]]和[[Replication Strategy (复制策略)|副本]]实现水平扩展和高可用。

---

## 🏗️ 核心架构

### 1. 集群 (Cluster)
- 一个或多个节点的集合
- 共享集群名称
- 提供索引和搜索能力

### 2. 节点 (Node)
- 一个 Elasticsearch 实例
- 有多种类型：
  - **Master-eligible Node**：可参与 Master 选举
  - **Data Node**：存储数据，执行 CRUD 和搜索
  - **Ingest Node**：数据预处理（管道）
  - **Coordinating Node**：协调节点，路由请求，聚合结果

### 3. 索引 (Index)
- 文档的逻辑集合
- 类似于数据库的库

### 4. 类型 (Type)
- 索引内的文档分类（7.x 后已废弃）

### 5. 文档 (Document)
- 最小数据单元
- JSON 格式
- 类似于数据库的行

### 6. 分片 (Shard)
- 索引的物理分片
- 每个分片是一个 Lucene 索引
- 分为：
  - **Primary Shard**：主分片，数据写入主分片
  - **Replica Shard**：副本分片，主分片的备份

---

## 📂 核心概念

### 倒排索引 (Inverted Index)
- **正向索引**：文档 → 词（如文档 1 包含词 A、B、C）
- **倒排索引**：词 → 文档（如词 A 出现在文档 1、2、3）
- 包含：
  - **词项 (Term)**：分词后的词
  - ** posting list**：词项出现的文档 ID 列表
  - **词频 (TF)**：词在文档中的出现次数
  - **文档频率 (DF)**：词出现的文档数

### 分词 (Analysis)
- 将文本切分为词项的过程
- 包含：
  - **Character Filters**：字符过滤（如去除 HTML）
  - **Tokenizer**：分词器（如按空格切分）
  - **Token Filters**：词项过滤（如小写、去停用词）

---

## 📝 写入流程

1. **协调节点接收请求**
   - 客户端请求任意节点
   - 该节点成为协调节点

2. **路由到主分片**
   - 计算文档应该存入哪个分片：`shard = hash(routing) % number_of_primary_shards`
   - 转发请求到对应主分片所在节点

3. **主分片写入**
   - 写入内存缓冲区 (In-memory Buffer)
   - 同时写入 [[WAL (预写式日志)|Translog]]（预写日志）

4. **生成 Segment**
   - 每隔 1 秒，内存缓冲区刷新到文件系统缓存 (Filesystem Cache)
   - 生成新的 Segment，此时文档可被搜索（近实时）
   - 但未刷盘

5. **刷盘 ([[Flush Strategy (刷盘策略)|Flush]])**
   - Translog 每 30 分钟或满时触发
   - Segment 刷入磁盘
   - 清空 Translog
   - 此时数据持久化

6. **副本同步**
   - 主分片写入后，并行写入所有副本分片
   - 所有副本写入成功后返回确认

---

## 🔍 读取流程

### 1. 查询阶段 (Query Phase)
- 协调节点将请求广播到所有分片（主或副本）
- 每个分片在本地执行查询，返回文档 ID 和相关度评分
- 协调节点收集结果，合并排序，取 Top N

### 2. 获取阶段 (Fetch Phase)
- 协调节点根据文档 ID 向对应分片请求完整文档
- 分片返回文档
- 协调节点组装结果返回给客户端

---

## 🔄 Segment 合并 (Merge)

### 为什么需要合并？
- Segment 是不可变的
- 删除和更新是追加标记（.del 文件）
- 过多小 Segment 会影响查询性能

### 合并策略
- Elasticsearch 自动在后台合并小 Segment 为大 Segment
- 合并时同时清理已删除文档
- 减少查询时需要查询的 Segment 数量

---

## ⚙️ 关键配置

### 集群配置
| 配置 | 说明 |
|-----|------|
| `cluster.name` | 集群名称 |
| `node.name` | 节点名称 |
| `node.master` | 是否可成为 Master |
| `node.data` | 是否存储数据 |
| `network.host` | 网络绑定地址 |

### 索引配置
| 配置 | 说明 |
|-----|------|
| `number_of_shards` | 主分片数（创建后不可改） |
| `number_of_replicas` | 副本数（可动态调整） |
| `refresh_interval` | 刷新间隔（默认 1s） |

---

## 🎯 场景与最佳实践

| 场景 | 建议 |
|-----|------|
| **日志分析** | 按日期分索引，使用 ILM 管理生命周期 |
| **全文搜索** | 合理设计 Mapping，选择合适分词器 |
| **高可用** | 每个索引至少 1 个副本，节点分布在不同机架 |
| **性能优化** | 分片数合理（通常 2-4 倍节点数），避免过多小索引 |

---

## 🔗 关联笔记

- [[写放大 vs 读放大]] — Elasticsearch 倒排索引的写放大和 Scatter-Gather 的读放大
- [[控制面 vs 数据面]] — Elasticsearch Master 控制面与 Data 数据面
- [[LSM-Tree (日志结构合并树)|LSM-Tree]] — Lucene Segment 合并与 LSM-Tree 的关系
- [[分布式设计术语]] — Primary/Replica、Quorum 等术语
- [[WAL (预写式日志)|WAL]] — Translog 预写日志
- [[Sharding & Partitioning (分片：分区)|分片]] — ES 分片机制的本质
- [[Controller Pattern (控制器模式)|控制器模式]] — ES Master Node 的控制器模式
