- **词根**：**Buffering (缓冲)** & **Pooling (池化)**
    
- **模式**：**Producer-Consumer Pattern with Intermediate Buffer (带中间缓冲的生产消费模式)**
    
- **核心价值**：
    
    1. **读写隔离 (Read-Write Isolation)**：利用 User-Space Memory 隔离 Kernel-Space IO Contention。
        
    2. **无锁化/少锁化**：将全局的大锁竞争，拆解为内存操作的小锁。
        
    3. **对象复用 (Object Reuse)**：减少 Direct Memory 的分配与回收开销（系统调用 overhead）。
- **应用**：
		1. rocketmq的transientStorePool
		2. 数据库：Log Buffer (Redo Log)
		3. 高性能网络库：**Netty(PooledDirectByteBuf)/ Disruptor(RingBuffer)**

---
## 🔗 关联笔记
- [[Double Buffering (双重缓冲)]] — 缓冲模式的进阶：双缓冲交换引用
- [[netty]] — Netty 中 Buffering & Pooling 的综合运用
- [[RocketMq持久化-commitlog中的优化]] — TransientStorePool 的实战应用
- [[分布式存储核心概念辨析手册]] — 刷盘策略中的缓冲机制
