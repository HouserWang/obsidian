1. **Clone 源码**：git clone https://github.com/apache/rocketmq.git
    
2. **Checkout 分支**：切到 release-4.9.4 (最经典稳定)。
    
3. **安装插件**：在 IDEA 里安装 **SequenceDiagram** 插件。
    
    - 用法：在 asyncPutMessage 方法上右键 -> Sequence Diagram。它能自动生成时序图，让你一眼看懂调用链路。
        
4. **寻找中文注释版**：
    
    - GitHub 上有很多大神维护的“带中文注释版 RocketMQ”。
        
    - 搜索关键词：RocketMQ 源码分析 或 RocketMQ-Annotation。
        
    - 推荐：可以直接搜 rocketmq-learning 类似的仓库，很多前辈已经把每一行都注好了。
5. **  
    不要逐行读**：千万不要从第一行读到最后一行，那是读小说的方法，不是读源码。
    
6. **带着问题读**：比如“我就想知道，Broker 收到消息后，是先写内存还是先写磁盘？”，然后利用 IDE 的 **Call Hierarchy (调用链)** 功能直奔主题。
    
7. **关注数据结构**：在读类之前，先看它的**成员变量 (Fields)**。
    
    - 比如看到 DefaultMessageStore 里有 CommitLog 和 ConsumeQueue，你就知道它是管存储的。
        
    - 比如看到 RebalanceImpl 里有 processQueueTable，你就知道它是管队列分配状态的。
        
8. **善用 Debug**：写一个最简单的 Demo（发一条消息），在上述核心方法打断点。**看 10 遍代码不如 Debug 跑 1 遍。**

这份清单涵盖了 **“大脑（NameServer）- 入口（Producer）- 心脏（Store）- 神经（Transaction/Delay）- 出口（Consumer）”** 五大板块。请按此顺序食用。

---

### 第一板块：大脑 —— NameServer (路由管理)
*虽然简单，但要懂它是怎么做到“无状态”和“高可用”的。*

**面试考点**：
*   Broker 挂了，NameServer 多久知道？（心跳机制）
*   NameServer 之间不通信，数据怎么保证一致？（CAP 中的 AP 设计）

**必读源码**：
1.  **`RouteInfoManager.java`** (核心管理类)
    *   **位置**：`namesrv/route`
    *   **关键方法**：
        *   `registerBroker`：Broker 心跳注册。看它怎么用 `ReadWriteLock` 更新内存 Map。
        *   `pickupTopicRouteData`：客户端拉取路由。
    *   **架构视角**：你会发现它没有任何持久化代码，全靠内存 HashMap。这就是它高性能的原因，但也意味着重启会丢数据（靠 Broker 重发心跳补回来）。

---

### 第二板块：入口 —— Producer (发送与容错)
*决定了消息能不能发出去，以及发给谁。*

**面试考点**：
*   发送消息时的负载均衡策略是什么？
*   **故障延迟机制（Latency Fault Tolerance）** 是怎么实现的？（高频难点）

**必读源码**：
1.  **`DefaultMQProducerImpl.java`**
    *   **位置**：`client/impl/producer`
    *   **关键方法**：`sendDefaultImpl`。发送流程的总指挥。
2.  **`MQFaultStrategy.java`** (故障熔断策略)
    *   **位置**：`client/latency`
    *   **关键方法**：`selectOneMessageQueue`。
    *   **架构视角**：重点看 `latencyFaultTolerance.isAvailable(brokerName)`。它实现了一个类似 Sentinel 的熔断逻辑：如果某 Broker 发送耗时高，未来几秒内就避开它。这是客户端高可用的核心。

---

### 第三板块：心脏 —— Broker Store (存储引擎)
*这是 RocketMQ 的半壁江山，也是与 Kafka 差异最大的地方。*

**面试考点**：
*   CommitLog 和 ConsumeQueue 的关系？（读写分离）
*   零拷贝是怎么落地的？
*   同步刷盘和异步刷盘的区别？

**必读源码**：
1.  **`CommitLog.java`** (数据文件)
    *   **位置**：`store`
    *   **关键方法**：`asyncPutMessage`。
    *   **架构视角**：看它如何加锁（SpinLock/ReentrantLock），如何调用 `MappedFile.appendMessage` 写内存，以及如何提交刷盘请求 `submitFlushRequest`。
2.  **`MappedFile.java`** (内存映射)
    *   **位置**：`store`
    *   **关键方法**：`init` (mmap), `appendMessagesInner`。
    *   **架构视角**：Java NIO `FileChannel.map` 的最佳实践。
3.  **`ReputMessageService.java`** (分发线程)
    *   **位置**：`store/DefaultMessageStore.java` 内部类
    *   **关键方法**：`doReput`。
    *   **架构视角**：这是“写快读也快”的秘密。它异步地把 CommitLog 里的数据解析出来，构建 `ConsumeQueue` 和 `IndexFile`。

---

### 第四板块：神经 —— 特色功能 (事务与延时)
*这是你最关心的部分，也是 RocketMQ 的差异化竞争力。*

#### 1. 分布式事务消息 (Transaction)
**面试考点**：
*   Half Message 是怎么存的？Consumer 为什么看不见？
*   事务回查（Check）是怎么触发的？
*   Commit/Rollback 发生了什么？

**必读源码**：
1.  **`SendMessageProcessor.java`** (Broker 端处理发送)
    *   **位置**：`broker/processor`
    *   **关键方法**：`asyncSendMessage`。
    *   **架构视角**：搜索 `parseHalfMessageInner`。看它怎么把 Topic 偷换成 `RMQ_SYS_TRANS_HALF_TOPIC`，并备份原 Topic。这是“半消息不可见”的根本原因。
2.  **`EndTransactionProcessor.java`** (处理提交/回滚)
    *   **位置**：`broker/processor`
    *   **关键方法**：`processRequest`。
    *   **架构视角**：
        *   **Commit**：把消息从 Half Topic 读出来，恢复原 Topic，重新写入 CommitLog（此时可见）。同时写一条 Op 消息（标记删除）。
        *   **Rollback**：直接写一条 Op 消息（标记删除）。
3.  **`TransactionalMessageCheckService.java`** (回查线程)
    *   **位置**：`broker/transaction`
    *   **关键方法**：`run` -> `onWaitEnd`。
    *   **架构视角**：看它如何遍历 Half Topic，对比 Op Topic，找出“超时未提交”的消息，然后发 RPC 给 Producer 去回查。

#### 2. 延时消息 (Delay/Schedule)
**面试考点**：
*   为什么不支持任意时间延时？
*   延时消息存在哪里？

**必读源码**：
1.  **`ScheduleMessageService.java`**
    *   **位置**：`store/schedule`
    *   **关键方法**：`start` (定时器), `computeDeliverTimestamp`。
    *   **架构视角**：看它如何把 Topic 偷换成 `SCHEDULE_TOPIC_XXXX`，并根据 `delayLevel` 存入不同的 Queue。

---

### 第五板块：出口 —— Consumer (重平衡)
*生产环境 80% 的问题（消息堆积、重复消费）都出在这里。*

**面试考点**：
*   Rebalance 是怎么做的？
*   并发消费（Concurrent）和顺序消费（Orderly）的实现区别？

**必读源码**：
1.  **`RebalanceImpl.java`** (重平衡算法)
    *   **位置**：`client/impl/consumer`
    *   **关键方法**：`rebalanceByTopic`。
    *   **架构视角**：看它如何根据 `cidAll` (所有消费者) 和 `mqAll` (所有队列) 进行分配。重点看 `AllocateMessageQueueAveragely`（平均分配算法）。
2.  **`PullMessageService.java`** (拉取线程)
    *   **位置**：`client/impl/consumer`
    *   **关键方法**：`run`。
    *   **架构视角**：RocketMQ 的 Push 模式，本质上是“长轮询 Pull”。这个线程不断从队列拿请求去拉消息。
3.  **`ConsumeMessageConcurrentlyService.java`** (并发消费)
    *   **位置**：`client/impl/consumer`
    *   **关键方法**：`submitConsumeRequest`。
    *   **架构视角**：看它怎么把拉到的消息丢进 `consumeExecutor` 线程池。这里决定了客户端的消费并发度。

---

### 资深工程师的“阅读顺序建议”

为了避免劝退，建议按这个顺序攻克：

1.  **热身**：`DefaultMQProducerImpl` (发送) -> `CommitLog` (存储)。**搞懂消息怎么落盘的。**
2.  **进阶**：`RebalanceImpl` (重平衡)。**搞懂消费者怎么分队列的。**
3.  **高阶**：`SendMessageProcessor` (事务消息逻辑) -> `ScheduleMessageService` (延时消息)。**搞懂 RocketMQ 的黑科技。**
4.  **补全**：`RouteInfoManager` (NameServer) -> `NettyRemotingServer` (通信)。**搞懂底层支撑。**

这次的清单补全了**事务消息**和**延时消息**，覆盖了 RocketMQ 最核心的 80% 源码。配合你的视频教程，效果会非常好！