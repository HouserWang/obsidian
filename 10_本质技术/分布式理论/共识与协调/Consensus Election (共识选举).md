# Consensus Election (共识选举)

### 📌 名词定义

Consensus Election（共识选举）是指**在多个对等节点中，通过一套明确的规则和协议，选出一个"领导者"的过程。当没有明确的主节点，或主节点失效时，分布式系统需要通过共识算法（如 Raft、Paxos、ZAB）在剩余节点中选举出新的主节点，以保证集群的可用性和一致性。**

### 🧠 第一性原理

"群龙不可无首。"
分布式系统需要一个 Leader 来协调写操作，避免冲突和[[Brain Split (脑裂)|脑裂]]。
**本质过程：通过投票和多数派确认，保证选举结果被所有节点接受。**

### 🛠️ 落地实现（典型流程）

```
Consensus Election 的核心流程：

1. Raft 选举流程
   - 所有节点初始为 Follower 状态
   - 超时未收到 Leader [[Heartbeat Mechanism (心跳机制)|心跳]] → 转为 Candidate
   - Candidate 发起投票请求（RequestVote RPC）
   - 获得多数派（N/2+1）投票 → 成为 Leader
   - Leader 定期发送心跳，维持权威

2. 任期（Term）机制
   - 每个任期以一次选举开始
   - 任期号单调递增，用于检测过时信息
   - 一个任期最多一个 Leader

3. 典型应用
   - etcd / Consul：使用 Raft 共识算法
   - ZooKeeper：使用 ZAB 协议
   - Kafka：依赖 ZooKeeper 或 KRaft 进行 Controller 选举

4. 关键特性
   - 选举安全性：一个任期最多一个 Leader
   - Leader 完整性：已提交的日志不会丢失
   - 多数派决策：保证选举结果的合法性
```

### ⚖️ 架构权衡

- **优点**：自动选举，无需人工干预；保证选举结果的一致性和安全性。
    
- **缺点**：选举期间集群不可用（无法处理写请求）；选举过程可能产生短暂的"群龙无首"状态。
    
- **大厂实践**：etcd、ZooKeeper、Kafka 都通过共识选举实现 Leader 管理，是分布式协调服务的核心机制。
