---
created: 2025-12-17 13:16
tags:
  - redis
  - reactor
  - io-multiplexing
  - brag-doc
category: Middleware
tech_stack:
  - Redis
  - Linux Kernel
status: 🟢 掌握
---

# 🚀 技术沉淀: Redis Cluster 读写请求全链路与 IO 模型演进

## 1. 第一性原理 (The "Why" & "How")
> 💡 **核心机制拆解**：从内核中断到应用层内存的完整漂流。

- **底层机制**：Smart Client 路由 + IO 多路复用 (epoll) + Reactor 模型 (单线程执行/多线程IO) + PageCache。
- **设计哲学**：**计算与 IO 分离** (Redis 6.0 核心)、**Append-Only** (顺序写盘)、**Gossip** (去中心化)。
- **全链路关键细节**：
    1. **连接建立**：内核三次握手 -> 全连接队列 -> 主线程 `epoll_wait` 捕获 Accept 事件 -> `accept()` 获取 FD -> 注册 Read 事件。
    2. **请求读取 (IO Threads)**：数据到达网卡 -> 内核 Buffer -> **IO 线程并行读取**并解析 RESP 协议 (解码) -> 放入主线程队列。(此时主线程忙轮询等待)
    3. **命令执行 (Main Thread)**：**主线程串行执行**命令 (内存操作，无锁，原子性) -> 写入 `AOF Buffer` (用户态) -> 写入 `Replication Buffer`。
    4. **持久化与复制**：
        - `write()` 系统调用将 AOF Buf 刷入 **Kernel PageCache** (此时未落盘) -> 后台线程 `fsync` 落盘。
        - 异步将 Replication Buf 发送给 Slave。
    5. **响应返回 (IO Threads)**：主线程将结果写入 Output Buffer -> **IO 线程并行**调用 `write()` 发送回网卡 -> Client。

## 2. 横向对比 (The Trade-off)
> ⚖️ **架构师视角**：没有最好的技术，只有最适合的权衡。

| 维度 | Redis (Remote Cache) | Caffeine (Local Cache) |
| :--- | :--- | :--- |
| **一致性** | **最终一致性** (通过延迟双删/主动刷新可控) | **弱一致性** (多实例间状态割裂，难以同步) |
| **IO 模型** | **IO 密集型** (受限于网络带宽与延迟) | **内存直读** (零网络开销，纳秒级) |
| **数据规模** | **海量** (集群分片，TB 级) | **受限** (受限于 JVM 堆内存，GB 级) |
| **适用场景** | 分布式锁、Session共享、核心业务数据 | 静态配置、极热点数据 (L1 缓存) |

- **核心差异点**：Redis 通过 **IO 多路复用** 解决了海量连接下的吞吐问题，适合作为分布式系统的**“共享内存”**；Caffeine 牺牲了数据一致性换取了**极致的读性能**，适合作为 Redis 前置的 L1 缓存 (多级缓存架构)。

## 3. B 端业务落地 (The Value)
> 💼 **场景化应用**：库存扣减、配置中心、分布式锁

- **痛点描述**：B 端业务逻辑复杂，直接操作 Redis API 容易导致 **缓存与 DB 不一致**，且难以统一处理 **防击穿/穿透** 逻辑，代码耦合度高。
- **解决方案**：
    1. 引入 **JetCache** 框架，统一封装 **Cache-Aside** 模式，利用 `@Cached` 注解实现业务代码解耦。
    2. 针对库存扣减场景，使用 **Lua 脚本** 封装 "查库存+扣减" 逻辑，在 Redis 主线程原子执行，避免并发超卖。
    3. 配置 **`penetrationProtect`** (互斥锁) 解决热点配置加载时的击穿问题。
- **收益/价值**：将缓存治理逻辑从业务代码中剥离，开发效率提升 30%；通过 Lua 脚本保证了库存操作的原子性，在无锁(分布式锁)的情况下实现了高并发扣减。

## 4. 简历话术预案 (Resume Snippet)
> 📝 **降维打击**：明年写简历时，直接复制这一段。

- **话术 1 (侧重原理)**：深入理解 Redis **Reactor 网络模型** 演进，掌握 **Redis 6.0 多线程 IO** 原理与 **PageCache** 落盘机制，能从 **内核与网络层** 分析 Redis 的性能瓶颈（如 BigKey 阻塞主线程 vs 阻塞 IO 线程的区别）。
- **话术 2 (侧重实战)**：设计了基于 **JetCache** 的多级缓存架构，利用 **Lua 脚本** 实现复杂的原子性业务操作（如库存扣减），解决了传统 Cache-Aside 模式下的 **并发数据不一致** 与 **缓存击穿** 问题。

---
## 🔗 关联知识
- [[Linux_IO_Models]]
- [[Redis_Persistence]]