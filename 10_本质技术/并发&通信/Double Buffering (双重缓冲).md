### 读写队列分离 (Read-Write List Separation)

**核心痛点**：刷盘线程（Flush）和 写入线程（Put）都要访问“写入请求队列”。如果直接用一个加锁的 List，刷盘慢的时候，写入就会被堵死。

#### 模式：双缓冲 / 读写分离

在 **GroupCommitService (同步刷盘服务)** 中，RocketMQ 使用了两个队列（容器）：requestsWrite 和 requestsRead。

- **写入时 (Producer 线程)**：
    
    - 只往 requestsWrite 里扔请求。
        
    - 锁竞争非常小（只锁 put 动作）。
        
- **处理时 (刷盘线程)**：
    
    - **关键动作：Swap (交换)**。
        
    - 刷盘线程醒来后，**加锁**，瞬间把 requestsWrite 和 requestsRead **交换引用**，然后**解锁**。
        
    - **效果**：Producer 可以继续往新的 requestsWrite 里扔数据，而刷盘线程慢慢处理 requestsRead 里的旧数据。**两者互不干扰，锁的持有时间微乎其微。**。   
  
 ==当你发现“生产者”和“消费者”因为抢同一个队列的锁而打架时，**搞两个队列，定时交换**，是解决并发冲突的神技。== 