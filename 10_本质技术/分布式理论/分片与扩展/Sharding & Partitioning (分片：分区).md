# Sharding / Partitioning (分片/分区)

### 📌 名词定义

Sharding / Partitioning（分片/分区）是指**将数据集水平切分，分布到不同节点的策略。是解决海量数据存储与计算的核心手段。每个节点只存储数据的一个子集，通过分片键（[[Partition Key & Data Skew (分区键与数据倾斜)|Partition Key]]）决定数据落在哪个分片。**

### 🧠 第一性原理

"分而治之。"  
单机容量有限，无法存储海量数据。  
**本质策略：将大数据集切分为多个小片段，分布到不同节点，突破单机容量和性能极限。**

### 🛠️ 落地实现（核心策略）

```
Sharding / Partitioning 的核心策略：

1. 基于哈希的分片（Hash Partitioning）
   - 对分片键计算哈希值：hash(key) % N
   - 分布均匀，但无法支持范围查询
   - 典型应用：Redis Cluster（哈希槽）、Cassandra

2. 基于范围的分片（Range Partitioning）
   - 按分片键的范围划分：1-1000万 → 分片1，1000万-2000万 → 分片2
   - 支持范围查询，但容易产生热点
   - 典型应用：MongoDB、HBase、TiDB

3. 地理位置分片（Geo Partitioning）
   - 根据用户地理位置将数据就近存储
   - 典型应用：CDN、全球化应用

4. 核心组件
   - 路由器（Router）：决定数据落在哪个分片
   - 协调者（Coordinator）：跨分片查询的协调节点
   - 元数据服务（Metadata Service）：存储分片映射关系

5. 核心挑战
   - [[Partition Key & Data Skew (分区键与数据倾斜)|数据倾斜]]：某些分片数据量远大于其他分片
   - 热点：某个分片访问频率极高
   - [[Rebalancing (再平衡)|再平衡]]：节点增删时需要迁移数据
   - 跨分片事务：最复杂的挑战之一
```

### ⚖️ 架构权衡

- **优点**：突破单机容量限制，水平扩展能力极强；分散负载，提升整体吞吐。
    
- **缺点**：增加系统复杂性；跨分片查询和事务性能极差；再平衡成本高。
    
- **大厂实践**：Kafka 的 Topic 分区、Redis Cluster 的哈希槽、Elasticsearch 的分片都是分片策略的经典实现。
