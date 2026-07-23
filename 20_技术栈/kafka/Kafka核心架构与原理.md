# Kafka 核心架构与原理

## 📌 第一性原理

**基于 [[Append-Only Log (仅追加日志)|Append-Only Log]] 的分布式消息队列**：通过顺序写磁盘和[[零拷贝|零拷贝]]实现高吞吐，通过[[Sharding & Partitioning (分片：分区)|分区]]和 [[ISR (同步副本列表)|ISR]] 实现高可用和水平扩展。

---

## 🏗️ 核心架构

### 1. 生产者 (Producer)
- 发送消息到 Broker
- 负责分区选择（Hash 或 Round-Robin）
- 支持批量发送和压缩

### 2. Broker (代理)
- Kafka 服务器节点
- 存储消息数据（Log Segment）
- 处理生产者和消费者请求
- 参与副本同步

### 3. 消费者 (Consumer)
- 从 Broker 拉取消息
- 维护消费进度 (Offset)
- 支持消费者组 (Consumer Group)

### 4. 消费者组 (Consumer Group)
- 一组消费者共同消费一个 Topic
- 每个 Partition 只能被组内一个消费者消费
- 实现负载均衡和容错

### 5. ZooKeeper / KRaft
- **ZooKeeper**：旧版依赖，存储元数据、Controller 选举
- **KRaft**：新版自协商，基于 Raft 共识算法，不再依赖 ZooKeeper

---

## 📂 核心概念

### 1. Topic (主题)
- 消息的逻辑分类
- 类似于数据库的表

### 2. Partition (分区)
- Topic 的物理分片
- 每个 Partition 是一个有序的 [[Append-Only Log (仅追加日志)|Append-Only Log]]
- 消息在 Partition 内有序，跨 Partition 无序
- 每个 Partition 有一个 Leader 和多个 Follower

### 3. Replica (副本)
- Partition 的冗余备份
- Leader 处理读写，Follower 同步数据
- ISR (In-Sync Replicas)：与 Leader 保持同步的副本列表，参考[[ISR (同步副本列表)|ISR]]

### 4. Offset (偏移量)
- 消息在 Partition 内的唯一标识
- 消费者通过 Offset 标记消费位置

---

## 📝 写入流程

1. **生产者发送消息**
   - 选择 Partition
   - 批量发送，压缩（可选）
   - 发送到 Leader Broker

2. **Leader 写入**
   - 追加到本地 Log（Page Cache）
   - 返回确认（根据 acks 配置）

3. **副本同步**
   - Follower 从 Leader 拉取消息
   - 写入本地 Log
   - 向 Leader 发送 ACK

4. **高水位更新**
   - Leader 收到 ISR 所有副本 ACK 后更新 HW (High Watermark)
   - HW 之前的消息对消费者可见

---

## 🔍 读取流程

1. **消费者请求**
   - 指定 Topic、Partition、Offset
   - 发送到 Leader Broker

2. **Broker 读取**
   - 从 Page Cache 读取（如果命中）
   - 使用 sendfile 零拷贝传输
   - 不经过 JVM 堆，直接从 Page Cache 到网卡

3. **消费者消费**
   - 处理消息
   - 提交 Offset（自动或手动）

---

## ⚙️ 关键配置

### 生产者配置
| 配置 | 说明 | 推荐值 |
|-----|------|--------|
| `acks` | 确认级别 | `all` (高可靠) / `1` (平衡) / `0` (高性能) |
| `retries` | 重试次数 | `3` |
| `batch.size` | 批量大小 | `16384` |
| `linger.ms` | 等待时间 | `10` |
| `compression.type` | 压缩类型 | `lz4` 或 `snappy` |

### Broker 配置
| 配置 | 说明 | 推荐值 |
|-----|------|--------|
| `log.retention.hours` | 消息保留时间 | 根据业务需求 |
| `log.segment.bytes` | 单个 Segment 大小 | `1073741824` (1GB) |
| `num.replica.fetchers` | 副本拉取线程数 | `2` |

### 消费者配置
| 配置 | 说明 | 推荐值 |
|-----|------|--------|
| `group.id` | 消费者组 ID | 必填 |
| `auto.offset.reset` | 无 Offset 时行为 | `latest` 或 `earliest` |
| `enable.auto.commit` | 自动提交 Offset | `false` (手动可控) |
| `max.poll.records` | 单次拉取最大消息数 | `500` |

---

## 🔄 副本同步机制

### [[ISR (同步副本列表)|ISR]] (In-Sync Replicas)
- 与 Leader 保持同步的副本列表
- 副本落后超过 `replica.lag.time.max.ms` 则被踢出 ISR
- 只有 ISR 中的副本能被选为新 Leader

### Leader 选举
- **旧版 (ZK)**：Controller 通过 ZooKeeper 协调选举
- **新版 (KRaft)**：基于 [[Consensus Election (共识选举)|Raft 共识算法]]，更快更稳定

---

## 🎯 场景与最佳实践

| 场景 | 配置建议 |
|-----|---------|
| **日志收集** | acks=1, 异步复制, 压缩 |
| **金融交易** | acks=all, min.insync.replicas=2, 同步复制 |
| **高吞吐** | 多 Partition, 批量发送, 压缩 |
| **顺序消费** | 单 Partition, 或单消费者消费 |

---

## 🔗 关联笔记

- [[中间件的读写流程]] — Kafka 与其他中间件的读写流程对比
- [[日志复制]] — Kafka ISR 机制在复制策略中的定位
- [[控制面 vs 数据面]] — Kafka KRaft 控制面与数据面分离
- [[写放大 vs 读放大]] — Kafka 顺序写与读放大分析
- [[零拷贝]] — Kafka sendfile 零拷贝机制
- [[分布式设计术语]] — ISR、Quorum、Append-Only Log 等术语
- [[Append-Only Log (仅追加日志)|Append-Only Log]] — Kafka 存储模型的核心
- [[Sharding & Partitioning (分片：分区)|分片]] — Kafka 分区机制的本质
- [[Replication Strategy (复制策略)|复制策略]] — Kafka ISR 在复制策略中的定位
