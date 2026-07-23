# Replica Set & Replication Factor (副本集与复制因子)

### 📌 名词定义

Replica Set（副本集）是指**一组存储相同数据的节点集合，包括一个主节点（Primary/Leader）和多个从节点（Secondary/Follower）。Replication Factor（复制因子）是指**数据的副本总数，决定了系统的数据冗余度和可用性。**

### 🧠 第一性原理

"鸡蛋放在多个篮子里。"
单点故障是分布式系统的最大威胁。
**本质设计：通过多副本冗余，保证即使部分节点故障，数据仍然可用。**

### 🛠️ 落地实现（典型应用）

```
Replica Set & Replication Factor 的核心应用：

1. 副本集架构
   - Primary（主节点）：处理所有写操作
   - Secondary（从节点）：异步或同步复制主节点数据
   - 读操作可以分发到 Secondary（读写分离）

2. 复制因子（Replication Factor）
   - RF=1：无冗余，节点故障数据丢失
   - RF=2：一个副本，允许挂一个节点
   - RF=3：两个副本，大厂标配（如 Kafka、Cassandra）
   - RF=5：四个副本，极高可用性（如金融系统）

3. 典型应用
   - Kafka：每个 Partition 有 RF 个 Replica，分布在不同的 Broker
   - Cassandra：每个 Keyspace 配置 RF，数据自动复制到 RF 个节点
   - HDFS：默认 RF=3，数据块复制到 3 个 DataNode

4. 副本放置策略
   - 同机房：低延迟，但无法容灾机房故障
   - 跨机房：容灾能力强，但延迟高、带宽成本高
   - 机架感知（Rack Awareness）：副本分布在不同的机架/机房
```

### ⚖️ 架构权衡

- **优点**：提高数据可用性和容错能力；支持读写分离，提升读性能。
    
- **缺点**：增加存储成本（RF 倍）；写入延迟增加（需要同步到多个副本）。
    
- **大厂实践**：Kafka、Cassandra、HDFS 都通过 RF=3 实现高可用，结合机架感知策略保证副本分布在不同故障域。
