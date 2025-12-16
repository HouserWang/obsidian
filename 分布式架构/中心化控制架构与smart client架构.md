

### 1. Apache Kafka
*   **【架构模式】**：**强中心化控制 (Strong Centralized Control) + 智能客户端 (Smart Client)**。
*   **【控制面：元数据管理】**：通过 ZK/KRaft 抢占选举出唯一的 **Controller**。Controller 负责生成和维护全网元数据（Metadata），并**指定** Partition Leader 的分配。
*   **【数据面：路由寻址】**：客户端 SDK (Smart Client) 定时拉取元数据并缓存。发送消息时，SDK 本地计算目标分区，**直连** Partition Leader 所在的 Broker。
*   **【一致性与故障转移】**：采用 **ISR (In-Sync Replicas)** 机制进行日志复制。Controller 监控 Broker 存活，一旦 Leader 宕机，Controller 从 ISR 列表中**指定**新的 Leader 并广播元数据更新，触发客户端重定向。

### 2. Apache RocketMQ
*   **【架构模式】**：**联邦注册/弱中心化 (Federated Registration) + 智能客户端 (Smart Client)**。
*   **【控制面：元数据管理】**：**NameServer** 为无状态注册中心，不参与选举决策。Broker 内部（如 DLedger 模式）通过 Raft **自选** Leader，然后将拓扑信息**注册**到 NameServer。NameServer 仅聚合路由表。
*   **【数据面：路由寻址】**：客户端 SDK (Smart Client) 定时轮询 NameServer 拉取路由表。发送消息时，SDK 本地执行负载均衡算法，**直连** Broker Master。
*   **【一致性与故障转移】**：标准版采用主从同步/异步复制，无自动故障转移（只读）。DLedger 模式采用 **Raft Log** 复制，Broker 组内**自动投票**选举新 Leader，并更新 NameServer 注册信息。

### 3. Redis Cluster
*   **【架构模式】**：**去中心化控制 (Decentralized Control) + 智能客户端 (Smart Client)**。
*   **【控制面：元数据管理】**：无中心节点。所有 Master 节点通过 **Gossip 协议** 交换状态，协商 **Slots (槽位)** 的分配信息。元数据（Slot Map）分散存储在每个节点。
*   **【数据面：路由寻址】**：客户端 SDK (Smart Client) 启动时拉取 Slot Map。操作时本地计算 CRC16 哈希值定位 Slot，**直连** 目标 Node。若路由错误，Node 返回 `MOVED` 指令纠正客户端。
*   **【一致性与故障转移】**：采用 **异步复制 (Async Replication)**，无 ISR 机制，故障时可能丢失数据。故障转移通过 **Failover 投票机制** 实现：Slave 发起选举，集群内其他 Master 投票，票数过半则晋升。

### 4. Redis Sentinel (哨兵)
*   **【架构模式】**：**强中心化控制 (Strong Centralized Control) + 客户端发现 (Client-Side Discovery)**。
*   **【控制面：元数据管理】**：Sentinel 节点间通过 Raft 选举出 **Sentinel Leader**。Leader 负责监控 Redis Master 状态，并维护当前合法的 Master 地址（元数据）。
*   **【数据面：路由寻址】**：客户端 SDK 连接 Sentinel 集群，**订阅**或查询 Master 地址变更。获取地址后，客户端**直连** Redis Master 进行操作。
*   **【一致性与故障转移】**：采用 **异步复制 (Async Replication)**，无 ISR 机制。故障转移由 Sentinel Leader **指定**：判定 Master 下线后，按照规则（优先级/偏移量）从 Slave 中提拔新 Master，并通知客户端。

### 5. Elasticsearch
*   **【架构模式】**：**强中心化控制 (Strong Centralized Control) + 服务端路由 (Server-Side Routing)**。
*   **【控制面：元数据管理】**：通过 Zen2/Raft 算法选举出唯一的 **Master Node**。Master 负责维护 **Cluster State**（包含路由表），并**指定** Primary Shard 的分配，将状态广播同步给所有节点。
*   **【数据面：路由寻址】**：客户端通常连接任意节点（非 Smart）。接收请求的节点充当 **Coordinating Node (协调节点)**，基于本地内存的 Cluster State 进行路由计算，将请求**转发**至目标 Data Node。
*   **【一致性与故障转移】**：采用 **PacificA (类 ISR)** 模型。Master 维护 **In-Sync Allocations** 列表。故障时，Master 从该列表中**指定** Replica 晋升为 Primary。写入需满足 `wait_for_active_shards` 确认。