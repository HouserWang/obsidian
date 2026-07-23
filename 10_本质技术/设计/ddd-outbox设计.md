在 DDD 架构中，**本地消息表（Outbox Pattern）**是解决“保持业务数据库操作与发送消息一致性（Atomicity）”的终极普适方案。

你说得非常对：RocketMQ 的事务消息确实提供了一条“技术捷径” [2]。但如果你使用的是 **Kafka、RabbitMQ、ActiveMQ** 等不支持内置事务消息的 MQ，你就必须自己在应用层或基础设施层落地 Outbox 模式。

在业界，针对非 RocketMQ 的场景，实现 Outbox 模式主要有以下三种级别的优雅落地方案：

---

### 一、 核心底线：Outbox 数据库表设计

在看方案之前，我们先设计一张极其通用的 `outbox_event` 表。无论用哪种方案，这张表的底层数据结构都是一致的：

```sql
CREATE TABLE `outbox_event` (
  `event_id` varchar(64) NOT NULL COMMENT '唯一事件ID',
  `aggregate_type` varchar(128) NOT NULL COMMENT '聚合根类型，如 Order',
  `aggregate_id` varchar(64) NOT NULL COMMENT '聚合根ID',
  `event_type` varchar(128) NOT NULL COMMENT '事件类型，如 OrderPaidEvent',
  `payload` text NOT NULL COMMENT '事件内容（JSON格式）',
  `status` varchar(32) NOT NULL DEFAULT 'PENDING' COMMENT '状态: PENDING/SENDING/SUCCESS/FAILED',
  `retry_count` int NOT NULL DEFAULT '0' COMMENT '重试次数',
  `created_time` datetime NOT NULL COMMENT '创建时间',
  `update_time` datetime NOT NULL COMMENT '更新时间',
  PRIMARY KEY (`event_id`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
**核心原则**：往这张表插入数据的操作，必须和修改业务数据的操作**运行在同一个本地数据库事务中（Atomic Commit）**。

---

### 二、 方案一：轻量级轮询方案（适合中小项目、无额外组件依赖）

这是最容易落地、最轻量级的做法，不需要引入任何外部中间件。它通过一个后台线程或定时任务去拉取（Pull）Outbox 表中的数据并发送给 MQ。

```text
  [业务事务开始]
    ├── 1. 修改业务数据 (UPDATE t_order ...)
    └── 2. 插入事件数据 (INSERT INTO outbox_event ...)
  [业务事务提交]
        │ (数据已安全落盘)
        v
  [定时任务 / 轮询线程] (e.g., XXL-JOB 或 Spring @Scheduled)
    ├── 3. 乐观锁锁定数据 (UPDATE outbox SET status='SENDING' WHERE status='PENDING')
    ├── 4. 读出锁定数据并发送到任意 MQ (Kafka / RabbitMQ)
    └── 5. 标记成功并删除/归档 (DELETE FROM outbox_event WHERE event_id = ?)
```

#### 🛠️ 优雅落地细节（如何避免扫表拖慢 DB？）：
1.  **抢占式乐观更新（防并发抢占）**：
    不要先 `SELECT` 再 `UPDATE`。为了防止多个任务节点重复拉取，应该先用一条带乐观锁的 SQL 去“强占”数据：
    ```sql
    UPDATE outbox_event 
    SET status = 'SENDING', update_time = NOW() 
    WHERE status = 'PENDING' 
    LIMIT 100; -- 每次只捞100条
    ```
2.  **消费发送**：
    只查询刚才被自己更新为 `SENDING` 的数据，多线程并发将其发送到 RabbitMQ 或 Kafka。
3.  **物理删除/历史归档**：
    发送成功后，直接 `DELETE` 掉该条记录，保持 `outbox_event` 表的体积处于极小的水平（只有几百或几千条延迟数据），**绝对不能让它无限膨胀，否则扫表会变成灾难**。

---

### 三、 方案二：标准企业级方案（CDC + 日志监听 - 极度推荐）

轻量级轮询方案虽然简单，但它有一个致命弱点：**定时轮询会导致数据库 CPU 损耗，且消息发送会有秒级的延迟。**

在企业级、大流量场景下，标准方案是 **CDC（Change Data Capture，变更数据捕获）**。

```text
  [业务事务] ──> 写入数据库 (业务表 + outbox_event 表)
                        │
                        v (写入 Binlog / WAL 物理日志)
                  [数据库 Binlog]
                        │
                        v (非阻塞、近乎零延迟监听)
                  [Debezium / Flink CDC] (CDC 工具)
                        │
                        v (解析解析并发送)
                  [Kafka / RabbitMQ]
```

#### 🛠️ 优雅落地细节：
1.  **利用 Debezium 或 Canal**：
    Debezium 是目前国际上最主流的开源 CDC 框架。它通过模拟成数据库的 Slave，实时、非阻塞地读取数据库的 **Binlog (MySQL)** 或 **WAL (PostgreSQL)**。
2.  **配置过滤规则**：
    配置 Debezium，**只监听 `outbox_event` 表的 `INSERT` 动作**。
3.  **流向 MQ**：
    Debezium 捕获到 `INSERT` 事件后，自动提取 `payload` 和 `event_type`，将其转化为消息直接推送到 Kafka 或 RabbitMQ。
4.  **优点**：
    *   **近乎零延迟**：毫秒级推送到 MQ。
    *   **对 DB 零压力**：它不需要写任何 SQL 去扫表，完全是通过读取操作系统的物理日志文件实现的，对数据库 CPU 没有损耗。
    *   **业务零感知**：应用层只需要把事件 `insert` 进表里，后面的发送工作完全由 CDC 中间件外包解决了。

---

### 四、 方案三：现代优雅方案（Spring 官方原生框架支持）

如果你使用的是 Spring Boot 生态，且希望代码写得极度优雅、高内聚，在 2026 年，最推荐的方案是使用 **Spring Modulith（Spring 模块化单体框架）** [3.2, 3.3]。

Spring 官方在 Spring Modulith 中内置了极其完美的 **Event Publication Registry（事件发布注册表，即 Outbox）** [3.1, 3.3]。

#### 🛠️ 优雅落地细节：
你只需要引入 Spring Modulith 的 Starter（比如 JDBC + RabbitMQ 适配器），你连 `outbox_event` 表都不需要自己建！

1.  **引入依赖**：
    ```xml
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-jpa</artifactId>
    </dependency>
    <!-- 引入 RabbitMQ 自动集成适配器 -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-events-rabbitmq</artifactId>
    </dependency>
    ```

2.  **应用层正常发布 Spring 事件**：
    你只需要在应用层像以前一样，调用 Spring 的 `ApplicationEventPublisher` 发布一个普通的本地事件：
    ```java
    @Transactional
    public void payOrder(OrderId orderId) {
        Order order = orderRepo.findById(orderId);
        order.pay();
        orderRepo.save(order);
        
        // 正常发布 Spring 本地事件
        eventPublisher.publishEvent(new OrderPaidEvent(order.getId()));
    }
    ```

3.  **Modulith 自动接管并变魔术**：
    *   Spring Modulith 检测到你在事务中发布了事件。它会拦截这个动作，**自动在本地事务中创建一张事件表，并把事件序列化存入该表** [3.1, 3.3]。
    *   事务成功提交后，它会**自动把事件推送到你配置的 RabbitMQ / Kafka 队列中** [3.1]。
    *   如果推送成功，自动在数据库里将该事件标记为 `Completed` [3.1]。
    *   如果推送失败（比如 RabbitMQ 挂了），在 RabbitMQ 恢复后，它会自动重新投递 [3.1]。

整个过程，你只写了最基础的本地事件发布代码，**其余的所有本地消息表、MQ 投递逻辑、防丢重试逻辑，全部由 Spring 官方框架在底层自动完成了。** 极其优雅 [3.1, 3.3]。

---

### ⚖️ 方案大PK与架构抉择

对于不使用 RocketMQ 的项目，你应该如何选择：

| 方案 | 适用场景 | 运维成本 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- | :--- |
| **方案一：轻量轮询** | 中小型项目、传统 MVC、不想引入任何新组件的项目。 | **极低** <br>(只需几行代码) | 零外部依赖，开发极快。 | 有轮询延迟，消耗 DB 性能。 |
| **方案二：CDC (Debezium)** | 大流量、高并发、微服务架构、对性能和延迟极度敏感的项目。 | **中等** <br>(需要搭建 Debezium/Kafka Connect) | 性能极致，毫秒级延迟，对业务数据库无任何压迫。 | 链路较长，需要维护 CDC 抽取集群。 |
| **方案三：Spring Modulith** | 使用 Spring Boot、追求优雅设计、单体或模块化单体架构。 [3.2, 3.3] | **极低** <br>(Spring 原生支持) | 极度优雅，代码侵入几乎为零，框架自动兜底。 [3.1, 3.3] | 强绑定 Spring 生态。 |

### ⚠️ 终极铁律：幂等性（Idempotency）
不管你采用上述哪种方案（包括 RocketMQ 事务消息），由于网络抖动和重试机制，**消息投递在分布式系统下只能保证“至少送达一次（At-Least-Once）”**。

因此，**消费者（Consumer）端必须实现幂等性校验（比如通过事件唯一 ID 做防重表，或数据库唯一索引）**，这才是让本地消息表完美闭环的最后一块拼图。



### 妥协三：利用 Spring 事务同步器（0 本地消息表，0 额外组件）

你没时间建 outbox_event 表，也没时间写定时任务，更没时间配 CDC。但你又极其担心：**“万一数据库事务回滚了，MQ 消息却发出去了（幻影消息），导致数据不一致怎么办？”**

#### 💡 最轻量的做法：使用 Spring 的 TransactionSynchronizationManager 监听事务提交 [3.4]

这是 Spring 生态里最被低估的“黑科技”，也是最轻量级、最完美的妥协方案 [3.4]。它可以确保：**只有当数据库事务 Commit 成功后，才会去发送 MQ 消息 [3.4]。**

code Java

```
@Service
public class OrderService {
    @Autowired private OrderMapper orderMapper;
    @Autowired private KafkaTemplate kafkaTemplate; // 或者是 RabbitTemplate

    @Transactional
    public void payOrder(Long orderId) {
        // 1. 执行数据库操作
        Order order = orderMapper.selectById(orderId);
        OrderDomain orderDomain = new OrderDomain(order);
        orderDomain.pay();
        orderMapper.updateById(orderDomain.getEntity());

        // 2. 极简妥协：注册事务同步器，不建表，不写定时任务
        // Spring 会在当前事务【成功 Commit 之后】，自动回调 afterCommit() 方法 [3.4]
        if (TransactionSynchronizationManager.isActualTransactionActive()) { // 确保有事务
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
                @Override
                public void afterCommit() {
                    // 只有数据库成功写完盘，才会执行这里！
                    // 彻底避免了“DB 回滚，MQ 却发出去了”的尴尬场景
                    kafkaTemplate.send("order-paid-topic", orderId.toString());
                }
            });
        }
    }
}
```

#### ⚖️ 这个妥协方案的代价是什么？（架构师心里必须有数）

- **优点**：
    
    - **开发成本为 0**：不用建表，不用写定时任务，不用引入任何第三方框架，只需要几行 Spring 原生代码 [3.4]。
        
    - **防回滚**：100% 保证了如果 DB 回滚，MQ 绝对不会发送 [3.4]。
        
- **缺点（极小概率事件）**：
    
    - 它**不是 100% 防丢**的。如果数据库刚刚 Commit 成功，还没来得及执行 afterCommit()，服务器突然**断电/宕机/被 Kill**，这条 MQ 消息就会丢失，且没有重试机制。
        
    - **大厂妥协共识**：对于 99% 的非金融核心账务系统（如：发个积分、推个通知、更新个缓存），这种因为“恰好在 Commit 后和发送前断电”导致丢消息的概率极低（可能一年碰不到一次）。相比于花两周时间去建本地消息表和配 CDC，**用这几行代码换取立刻上线，是性价比高到爆炸的妥协方案！**