# AP System (AP 系统)

### 📌 名词定义

AP 系统是指**在 [[CAP Theorem (CAP 定理)|CAP 定理]] 的约束下，选择可用性（Availability）和分区容错性（Partition Tolerance），放弃[[Strong Consistency & Linearizability (强一致性与线性化)|强一致性]]（Consistency）的分布式系统。当发生网络分区时，AP 系统会继续提供服务，但允许数据存在短暂的不一致。**

### 🧠 第一性原理

"服务永远在线，数据最终一致。"  
对于高并发、大流量的互联网应用，用户体验和系统可用性比即时的数据一致性更重要。  
**本质选择：在"数据延迟"和"服务中断"之间，选择前者。**

### 🛠️ 落地实现（典型代表）

```
AP 系统典型代表：
├── Kafka (分布式消息队列)
│   └── [[ISR (同步副本列表)|ISR]] 机制，允许副本短暂不一致
├── Cassandra (分布式数据库)
│   └── 无主复制，通过 [[Quorum (法定人数)|Quorum]] 和 [[Read Repair & Hinted Handoff (读修复与提示移交)|Read Repair]] 保证[[Eventual Consistency (最终一致性)|最终一致]]
├── Redis Cluster (分布式缓存)
│   └── 异步复制，默认读 Master 保证一致性，可选读 Slave
└── RocketMQ (消息队列)
    └── 异步复制，追求极致吞吐

共同特征：
- 写操作本地成功即返回，异步复制到副本
- 网络分区时，所有节点继续提供服务
- 通过[[Eventual Consistency (最终一致性)|最终一致性]]（Eventual Consistency）保证数据最终正确
- 性能极高，适合高并发场景
```

### ⚖️ 架构权衡

- **优点**：高可用性、高吞吐量，网络分区时仍能提供服务。
    
- **缺点**：存在数据延迟，可能读到过期数据（Stale Read）。
    
- **适用场景**：互联网应用、消息队列、缓存系统、日志收集等高并发场景。
