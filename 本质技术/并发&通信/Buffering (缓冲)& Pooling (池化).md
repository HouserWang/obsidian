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

