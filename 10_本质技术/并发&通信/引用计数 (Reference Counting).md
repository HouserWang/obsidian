**核心痛点**：MappedFile 是一个文件映射。如果后台线程正在把这个文件删掉（文件过期），而前台线程正在读这个文件，怎么防止崩盘（Crash）？

#### 模式：Atomic 引用计数

这是 C++ 智能指针的思想，被搬到了 Java 里。

- **类**：ReferenceResource (MappedFile 的父类)。
    
- **机制**：
    
    - AtomicInteger refCount：记录有多少人在用这个文件。
        
    - **使用前 (hold)**：refCount.getAndIncrement()。
        
    - **使用后 (release)**：refCount.getAndDecrement()。
        
    - **删除时 (shutdown)**：先标记为不可用，然后检查 refCount 是否为 0。如果不为 0，**坚决不删**，等着。
      

==在管理"外部资源"（文件、连接、堆外内存）时，**引用计数**是防止"野指针"和"资源泄露"的唯一真理。==

---
## 🔗 关联笔记
- [[RocketMq持久化-commitlog中的优化]] — MappedFile 引用计数的实战场景
- [[预热与预分配]] — MappedFile 的生命周期管理
- [[Double Buffering (双重缓冲)]] — 读写队列交换时的并发安全
- [[零拷贝]] — mmap 映射文件的管理
