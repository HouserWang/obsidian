# Read Repair & Hinted Handoff (读修复与提示移交)

### 📌 名词定义

Read Repair（读修复）是指**在读取数据时，发现副本间数据不一致，主动触发修复过程，将最新数据同步到过期副本。Hinted Handoff（提示移交）是指**写入时目标节点宕机，将数据和"提示"交给其他节点暂存，待目标节点恢复后转交。**两者都是 [[AP System (AP 系统)|AP 系统]]保证[[Eventual Consistency (最终一致性)|最终一致性]]的后台修复机制。**

### 🧠 第一性原理

"读时顺便修，写时帮忙存。"  
在去中心化的分布式系统中，无法保证每次写入都同步到所有副本。  
**本质策略：通过读操作和临时存储，弥补异步复制带来的不一致。**

### 🛠️ 落地实现（典型应用）

```
Read Repair & Hinted Handoff 的核心机制：

1. Read Repair（读修复）
   - 客户端读取数据，向多个副本发起请求
   - 比较各副本返回的数据版本（通过[[Vector Clock (向量时钟)|向量时钟]]或时间戳）
   - 发现不一致时，将最新数据写回过期副本
   - 典型应用：Cassandra、DynamoDB

2. Hinted Handoff（提示移交）
   - 写入时目标节点 A 宕机
   - 将数据写入节点 B，并在 B 中记录一条"提示"：这条数据属于 A
   - 节点 A 恢复后，节点 B 将数据转交给 A，并删除提示
   - 典型应用：Cassandra、Riak

3. 协同工作
   - 写入时：目标节点宕机 → Hinted Handoff 暂存
   - 读取时：发现副本不一致 → Read Repair 修复
   - 两者结合，保证数据最终一致

4. 典型场景
   - Cassandra 的 [[Quorum (法定人数)|QUORUM]] 读写
   - 写入 3 个副本中的 2 个（W=2, R=2, N=3）
   - 读取时发现第 3 个副本数据过期 → Read Repair
```

### ⚖️ 架构权衡

- **优点**：无需中心协调，自动修复数据不一致；Hinted Handoff 提高写入可用性。
    
- **缺点**：Read Repair 增加读取延迟；Hinted Handoff 需要额外的存储空间和清理机制。
    
- **大厂实践**：Cassandra 通过 Read Repair + Hinted Handoff 实现高可用的最终一致性，无需任何中心节点。
