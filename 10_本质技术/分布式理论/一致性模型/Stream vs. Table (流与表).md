# Stream vs. Table (流与表)

### 📌 名词定义

Stream 与 Table 是**数据动态性的两种互补视角。Stream 是无限的事件序列，关注"变化"；Table 是某一时刻的状态快照，关注"当前值"。两者可互相转换：Table 是 Stream 的[[物化视图|物化视图]]，Stream 是 Table 的变更日志。**

### 🧠 第一性原理

"数据是一条河，还是一潭水？"  
所有数据本质上都是事件流（Stream），数据库表（Table）只是流在某一时刻的快照。  
**本质洞察：理解 Stream 与 Table 的等价性，就理解了 CDC、流处理和事件溯源的根源。**

### 🛠️ 落地实现（典型应用）

```
Stream 与 Table 的转换：

1. Stream → Table（[[物化视图|物化视图]]）
   - Kafka Topic (Stream) → 数据库汇总表 (Table)
   - 事件流：OrderCreated, OrderPaid, OrderShipped
   - 物化后：订单表当前状态 = {status: "已发货"}

2. Table → Stream（变更日志/CDC）
   - MySQL Binlog (Stream) ← 数据库表 (Table)
   - 每次 UPDATE/INSERT 都是 Stream 中的一个事件
   - Debezium 通过监听 Binlog 捕获 Table 的变更流

3. 核心抽象
   - Kafka：以 Stream 为核心抽象，消息是事件
   - 数据库：以 Table 为核心抽象，数据是当前状态
   - Kafka Streams / Flink：在 Stream 上做 Table 的物化

4. 典型应用
   - CDC (Change Data Capture)：捕获 Table 变更生成 Stream
   - 事件溯源 (Event Sourcing)：只存 Stream，Table 是派生的
   - CQRS：写端用 Stream，读端用 Table
```

### ⚖️ 架构权衡

- **优点**：理解 Stream 与 Table 的等价性，有助于设计更灵活的数据架构（如事件溯源、CQRS）。
    
- **缺点**：Stream 模型需要处理无限序列、乱序事件、重复事件等复杂性。
    
- **大厂实践**：Kafka 以 Stream 为中心，数据库以 Table 为中心，CDC 工具（Debezium）连接两者，实现数据的实时同步。
