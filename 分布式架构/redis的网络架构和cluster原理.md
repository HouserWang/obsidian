### 第一部分：Redis 网络架构（单机高性能的秘密）

很多初级工程师认为“Redis 是单线程的”。作为资深人士，你必须精确地描述：**Redis 的网络模型经历了从“纯单线程”到“多线程 IO”的演变。**

#### 1. 核心原理：IO 多路复用 (IO Multiplexing)

这是 Redis 高性能的基石。Redis 不会为每个连接开一个线程（那样 Context Switch 会杀得 CPU 只有 50% 利用率），而是利用操作系统提供的 **epoll (Linux)** / **kqueue (BSD/Mac)** 机制。

- **Reactor 模式：** Redis 是典型的 **Event Loop (事件循环)** 架构。
    
- **流程：**
    
    1. Kernel 监听成千上万个 Socket 连接。
        
    2. 一旦某个 Socket 有数据来了（Readable），Kernel 通知 Redis。
        
    3. Redis 主线程把这个事件放入队列，逐个处理。
        

#### 2. Redis 4.0/5.0：单 Reactor 单线程

- **架构：** 接收连接、读取 Socket、解析命令、执行内存操作、写回 Socket，**全都在一个主线程里串行完成**。
    
- **为什么快？**
    
    - 纯内存操作（纳秒级）。
        
    - 没有线程切换损耗。
        
    - 没有锁竞争（不需要考虑并发安全）。
        
- **瓶颈：** **CPU 不是瓶颈，网络 IO 才是。** 当 Value 很大（比如 100KB），主线程在 write() 发送数据时，会占用大量 CPU 时间，导致后面的请求排队。
    

#### 3. Redis 6.0+：单 Reactor 多线程 (IO Threading)

这是重大变革。为了解决网络 IO 的瓶颈，Redis 引入了 **多线程处理网络读写**，但**命令执行依然是单线程**。

- **架构流程：**
    
    1. **Main Thread:** 获取活跃的 Socket 连接。
        
    2. **IO Threads (多线程):** 并行地从 Socket **读取**数据，并**解析**命令。（此时主线程忙别的，或等待）。
        
    3. **Main Thread:** 等大家读完了，**串行执行命令**（SET/GET...）。**注意：这里依然是原子的，不需要锁！**
        
    4. **IO Threads (多线程):** 并行地把结果 **写入** Socket 发给客户端。
        
- **总结：** **“计算单线程，IO 多线程”**。既保留了无锁的优势，又吃满了网卡带宽。
    

---

### 第二部分：Redis Cluster 架构原理（去中心化的艺术）

我们在上一个问题聊到了 Cluster 没有 Controller，现在深入其**拓扑结构**和**数据分布算法**。

#### 1. 拓扑结构：全网状 (Full Mesh)

- **架构：** 如果集群有 N 个节点，每个节点都和其他 N-1 个节点保持 TCP 长连接。
    
- **端口：**
    
    - 6379: 正常服务端口。
        
    - 16379 (6379+10000): **Cluster Bus (集群总线)** 端口。
        
- **通信内容：** 专门跑 **Gossip 协议**。
    

#### 2. 数据分片：Hash Slot (哈希槽)

Redis Cluster 没有用一致性哈希，而是用了 **Hash Slot**。

- **总量：** **16384** 个槽（0 ~ 16383）。
    
- **算法：** Slot = CRC16(key) % 16384。
    
- **为什么是 16384？（面试高频）**
    
    - Redis 节点之间发心跳包时，会携带一个 bitmap 来表示“我负责哪些槽”。
        
    - 如果槽太多（比如 65536），心跳包太大（8KB），容易造成网络拥堵。
        
    - 如果槽太少，数据切分不均。
        
    - 16384 (2KB bitmap) 是作者 Antirez 在**网络带宽**和**分片粒度**之间做的 Trade-off。
        

#### 3. 客户端路由：MOVED 与 ASK

这是 Smart Client 的核心逻辑。

- **MOVED (永久重定向):**
    
    - 客户端访问 Node A：GET key1。
        
    - Node A 计算发现 key1 的 Slot 在 Node B。
        
    - Node A 返回：-MOVED 3999 192.168.1.2:6379。
        
    - 客户端行为：更新本地路由表（以后 key1 所在的 Slot 直接找 Node B），并重试 Node B。
        
- **ASK (临时重定向 - 正在迁移中):**
    
    - 场景：Slot 正在从 A 搬家到 B。Key1 可能还在 A，也可能跑到了 B。
        
    - 客户端访问 Node A。Node A 发现 Slot 正在迁移且 Key1 不在 A 了，返回 -ASK ...。
        
    - 客户端行为：**只针对这一次请求**去访问 Node B（先发 ASKING 命令，再发 GET）。**不更新**本地路由表。
        

#### 4. 故障发现与 Failover (Gossip 的力量)

没有 ZK，怎么知道谁挂了？

1. **PFAIL (Subjective Fail - 主观下线):** Node A 给 Node B 发 Ping，B 超时没回。A 在心里给 B 打个标记：“我觉得 B 挂了”。
    
2. **Gossip 传播:** A 遇到 C，告诉 C：“我觉得 B 挂了”。
    
3. **FAIL (Objective Fail - 客观下线):** 当集群中 **半数以上的主节点** 都告诉 A 说“我也觉得 B 挂了”，A 就会正式广播：**“B 确诊已挂！”**。
    
4. **从节点选举:** B 的从节点们（Slave B1, Slave B2）开始竞选。
    
    - 利用 Raft 类似的协议（Epoch 纪元），数据最新的 Slave 优先发起投票。
        
    - 其他 Master 节点进行投票。
        
    - 票数过半，B1 当选新 Master，接管 Slot。