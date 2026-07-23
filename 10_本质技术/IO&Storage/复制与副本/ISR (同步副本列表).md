# ISR (同步副本列表)

### 📌 名词定义

ISR (In-Sync Replicas，同步副本列表) 是指**Kafka 动态维护的健康副本列表，保证可用性和一致性。只有 ISR 中的副本才有资格成为 Leader，写操作必须得到 ISR 中所有副本（或配置的 min.insync.replicas）的确认。**

### 🧠 第一性原理

"动态 Quorum，踢掉慢节点。"
传统 [[Quorum (法定人数)|Quorum]] 机制（如 Raft）要求固定的多数派节点确认，慢节点会拖累整体性能。
**本质创新：通过动态维护健康副本列表，踢掉慢节点，在保证一致性的同时提升可用性。**

### 🛠️ 落地实现（核心机制）

```
ISR 的核心机制：

1. ISR 维护
   - 每个 Partition 有一个 ISR 列表
   - Follower 必须及时同步 Leader 的数据（在 replica.lag.time.max.ms 内）
   - 同步及时的 Follower 留在 ISR 中
   - 同步延迟的 Follower 被踢出 ISR

2. 写操作确认
   - acks=1：Leader 写入即返回
   - acks=all：ISR 中所有副本确认才返回
   - min.insync.replicas：ISR 中最少需要同步的副本数
   - 如果 ISR 大小 < min.insync.replicas，拒绝写入

3. Leader 选举
   - Leader 故障时，从 ISR 中选择新的 Leader
   - 由 [[Controller Pattern (控制器模式)|Controller]] 指定，无需共识选举
   - 保证新 Leader 拥有最新数据

4. 典型应用
   - Kafka：每个 Partition 维护 ISR 列表
   - Elasticsearch：类似 ISR 的 In-Sync Allocations
   - HDFS：类似 ISR 的 Pipeline 复制

5. 与 Raft 的对比
   - Raft：固定 [[Quorum (法定人数)|Quorum]]（N/2+1），慢节点拖累性能
   - ISR：动态 Quorum，踢掉慢节点，性能更好
   - Raft：强一致（CP），ISR：偏 AP（通过配置可达 CP）
```

### ⚖️ 架构权衡

- **优点**：动态踢掉慢节点，性能优于固定 Quorum；可用性极高，3 个节点挂 2 个仍能写入。
    
- **缺点**：ISR 维护增加复杂性；如果 ISR 列表过小，可能丢失数据。
    
- **大厂实践**：Kafka 通过 ISR 机制实现高可用和高性能的数据复制；Elasticsearch 通过类似的 In-Sync Allocations 保证 Shard 的一致性。
