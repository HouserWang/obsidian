# Replication Strategy (复制策略)

### 📌 名词定义

Replication Strategy（复制策略）是指**数据何时同步到其他节点的分布式持久化策略。包括同步复制（等待所有副本确认）、异步复制（本地成功即返回，后台同步副本）和半同步复制（至少一个副本确认）。是 [[State Machine Replication (状态机复制)|State Machine Replication]] 的具体实现机制。**

### 🧠 第一性原理

"数据什么时候同步到副本？"  
本地持久化（刷盘）只保证单机数据安全，分布式系统需要多副本冗余。  
**本质权衡：同步复制安全但慢，异步复制快但可能丢数据，半同步复制是折中方案。**

### 🛠️ 落地实现（核心策略）

```
Replication Strategy 的核心策略：

1. 同步复制（Sync Replication）
   - 写入本地后，等待所有副本确认才返回
   - 优点：数据极安全，所有副本一致
   - 缺点：性能极差，延迟高
   - 典型应用：金融系统、RocketMQ SYNC_MASTER

2. 异步复制（Async Replication）
   - 写入本地后立即返回，后台异步同步副本
   - 优点：性能极高，延迟低
   - 缺点：主节点故障时可能丢数据
   - 典型应用：MySQL 默认、Redis 默认、RocketMQ ASYNC_MASTER

3. 半同步复制（Semi-Sync Replication）
   - 写入本地后，至少等待一个副本确认才返回
   - 优点：比异步安全，比同步性能好
   - 缺点：仍有数据丢失风险（确认的副本也可能故障）
   - 典型应用：MySQL 半同步（大厂标配）

4. [[ISR (同步副本列表)|ISR]] 机制（动态 [[Quorum (法定人数)|Quorum]]）
   - 维护健康副本列表，踢掉慢节点
   - 写操作需 ISR 中所有副本确认
   - 典型应用：Kafka、Elasticsearch

5. 与其他概念的关系
   - 复制策略是 [[State Machine Replication (状态机复制)|State Machine Replication]] 的具体实现机制
   - [[Flush Strategy (刷盘策略)|刷盘策略]]（本地）+ 复制策略（分布式）= 完整的持久性保证
   - 同步复制向 CP 靠拢，异步复制偏 AP

6. 典型应用
   - MySQL：异步 / 半同步 / MGR（基于 Paxos）
   - Redis：异步复制，可选 WAIT 命令提升一致性
   - Kafka：ISR 机制，acks 参数控制一致性
   - RocketMQ：ASYNC_MASTER / SYNC_MASTER
```

### ⚖️ 架构权衡

- **优点**：同步复制保证数据不丢失；异步复制性能极高；半同步复制是折中方案。
    
- **缺点**：同步复制性能差；异步复制可能丢数据；半同步复制仍有风险。
    
- **大厂实践**：MySQL 半同步是大厂标配；Kafka 通过 ISR + acks 实现可调一致性；RocketMQ 提供 ASYNC_MASTER 和 SYNC_MASTER 两种选择。
