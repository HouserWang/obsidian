# CAP Theorem (CAP 定理)

### 📌 名词定义

CAP 定理是指**在一个[[分布式事务|分布式系统]]中，一致性（Consistency）、可用性（Availability）、分区容错性（Partition Tolerance）三者不可兼得，最多只能同时满足其中两项。**

### 🧠 第一性原理

分布式系统的物理本质。当节点之间存在网络通信时，网络分区（Network Partition）是不可避免的物理现实。  
**本质约束：在发生网络分区时，系统必须在"拒绝服务（保一致）"和"继续服务（保可用）"之间做出选择。**

### 🛠️ 落地实现（三种架构选择）

```
┌─────────────────────────────────────────────────────────┐
│                    CAP Theorem                           │
├─────────────────────────────────────────────────────────┤
│  CA (Consistency + Availability)                        │
│  - 传统单节点数据库（非分布式）                          │
│  - 无分区容错，一旦网络分区就不可用                      │
│                                                         │
│  CP (Consistency + Partition Tolerance)                 │
│  - [[CP System (CP 系统)|Etcd]], ZooKeeper, TiKV                                │
│  - 网络分区时，少数派节点拒绝服务                       │
│  - 保证已确认的数据绝不丢失                             │
│                                                         │
│  AP (Availability + Partition Tolerance)                │
│  - Kafka, Cassandra, Redis Cluster                      │
│  - 网络分区时，所有节点继续服务                         │
│  - 允许短暂的数据不一致                                 │
└─────────────────────────────────────────────────────────┘
```

### ⚖️ 架构权衡

- **优点**：CAP 定理是架构师设计分布式系统的根本约束，帮助快速定位系统的核心设计目标。
    
- **缺点**：CAP 是一个过于简化的模型，实际系统往往是在一致性和可用性之间做更细粒度的权衡（如 [[Tunable Consistency (可调一致性)|Tunable Consistency]]）。
    
- **大厂实践**：现代分布式系统通常采用"分区正常时追求[[Strong Consistency & Linearizability (强一致性与线性化)|强一致]]，分区异常时降级为[[Eventual Consistency (最终一致性)|最终一致]]"的策略。
