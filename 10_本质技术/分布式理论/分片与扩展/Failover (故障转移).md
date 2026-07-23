# Failover (故障转移)

### 📌 名词定义

Failover（故障转移）是指**当主节点（Leader/Master）发生故障时，系统自动或手动将服务切换到备用节点（Follower/Slave）的过程。是分布式系统高可用性的核心保障机制。**

### 🧠 第一性原理

"主节点挂了，备节点顶上。"
分布式系统中，节点故障是常态而非例外。
**本质策略：通过冗余副本和自动切换机制，保证单点故障不影响整体服务。**

### 🛠️ 落地实现（典型模式）

```
Failover 的核心模式：

1. 自动故障转移（共识算法）
   - Raft/ZAB：[[Leader Election (领导者选举)|Leader]] 故障后，剩余节点自动选举新 Leader
   - 秒级切换，无需人工干预
   - 典型应用：etcd、ZooKeeper、Kafka KRaft

2. 半自动故障转移（外部工具）
   - MySQL MHA / Orchestrator：监控主库，故障时自动切换
   - Redis Sentinel：Sentinel Leader 判定 Master 下线，提拔新 Master
   - 需要外部工具辅助，切换时间较长

3. 手动故障转移
   - 传统主从复制：DBA 手动切换主从角色
   - 风险高，切换时间长，可能丢失数据

4. 关键指标
   - RTO (Recovery Time Objective)：恢复时间目标
   - RPO (Recovery Point Objective)：恢复点目标（允许丢失的数据量）
   - 共识算法：RTO 秒级，RPO = 0（零丢失）
   - 异步复制：RTO 分钟级，RPO > 0（可能丢失数据）
```

### ⚖️ 架构权衡

- **优点**：保证系统高可用性，单点故障不影响整体服务。
    
- **缺点**：自动切换可能产生误判（网络分区时）；手动切换风险高、时间长。
    
- **大厂实践**：etcd/ZooKeeper 通过共识算法实现自动故障转移；MySQL/Redis 通过外部工具实现半自动切换；传统数据库依赖 DBA 手动切换。
