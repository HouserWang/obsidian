# Leader Election (领导者选举)

### 📌 名词定义

Leader Election（领导者选举）是指**在分布式系统中，通过共识算法（如 Raft、Paxos、ZAB）从多个对等节点中选出一个"领导者"的过程。Leader 负责协调写操作，避免冲突和[[Brain Split (脑裂)|脑裂]]。当 Leader 故障时，剩余节点自动选举出新的 Leader。**

### 🧠 第一性原理

"一山不容二虎，但必须有一只虎。"  
分布式系统需要一个中心节点来协调写操作，避免多主冲突。  
**本质过程：通过投票和多数派确认，保证选举结果被所有节点接受，且在 Leader 故障时能自动切换。**

### 🛠️ 落地实现（典型应用）

```
Leader Election 的核心应用：

1. Raft 选举
   - 所有节点初始为 Follower
   - 超时未收到 Leader 心跳 → 转为 Candidate
   - Candidate 发起投票，获得多数派支持 → 成为 Leader
   - Leader 定期发送心跳，维持权威

2. ZooKeeper (ZAB)
   - 类似 Raft，但使用 ZAB 协议
   - 选举出 Leader 后，Leader 负责协调所有写操作

3. Kafka Controller
   - 通过 ZooKeeper 或 KRaft 选举出唯一的 Active Controller
   - Controller 负责 Partition Leader 的分配和切换

4. 关键特性
   - 选举安全性：一个任期最多一个 Leader
   - Leader 完整性：已提交的日志不会丢失
   - 自动故障转移：Leader 故障后自动选举新 Leader
```

### ⚖️ 架构权衡

- **优点**：自动选举，无需人工干预；保证选举结果的一致性和安全性；Leader 故障时自动切换。
    
- **缺点**：选举期间集群不可用（无法处理写请求）；选举过程可能产生短暂的"群龙无首"状态。
    
- **大厂实践**：etcd、ZooKeeper、Kafka 都通过 Leader 选举实现分布式协调，是分布式系统的核心机制。
