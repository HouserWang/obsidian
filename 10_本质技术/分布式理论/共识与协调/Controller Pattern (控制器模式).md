# Controller Pattern (控制器模式)

### 📌 名词定义

Controller Pattern（控制器模式）是指**由一个中心化的"大脑"管理集群元数据和调度的架构模式。这个控制器通过共识算法（如 Raft、ZAB）实现高可用，负责决定 Partition 的分布、Leader 的身份、Shard 的分配等关键决策。**

### 🧠 第一性原理

"一个集群需要一个大脑。"
分布式系统需要一个中心化的组件来维护全局视图，做出关键决策。
**本质设计：将"决策"与"执行"分离，控制器负责决策，数据节点负责执行。**

### 🛠️ 落地实现（典型应用）

```
Controller Pattern 的核心应用：

1. Kafka Controller
   - 通过 KRaft/ZK 选举出唯一的 Active Controller
   - 负责 Partition Leader 的分配和切换
   - 维护集群元数据（Topic、Partition、Replica 分布）
   - 当 Broker 宕机时，Controller 指定新的 Leader

2. Kubernetes Controller Manager
   - 多个 Controller 共同管理集群状态
   - Deployment Controller：管理 Pod 副本数
   - ReplicaSet Controller：保证 Pod 副本数符合预期
   - Node Controller：监控节点健康状态

3. Elasticsearch Master Node
   - 通过 Raft 选举出唯一的 Master Node
   - 负责维护 Cluster State（路由表）
   - 决定 Shard 在 Data Node 上的分配
   - 当 Data Node 宕机时，Master 重新分配 Shard

4. 共同特征
   - 控制器本身通过共识算法保证高可用
   - 控制器不处理用户请求，只负责元数据管理
   - 数据面在控制器的"钦定"下运行轻量级同步
```

### ⚖️ 架构权衡

- **优点**：集中管理，决策高效；控制器本身通过共识算法保证高可用；数据面无需运行重型共识协议。
    
- **缺点**：控制器可能成为性能瓶颈；控制器故障时需要重新选举，期间集群无法做决策。
    
- **大厂实践**：Kafka、K8s、Elasticsearch 都采用 Controller Pattern，将"共识"集中在控制面，释放数据面的极致吞吐。
