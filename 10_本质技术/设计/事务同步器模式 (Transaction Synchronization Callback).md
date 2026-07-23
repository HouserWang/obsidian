# 事务同步器模式 (Transaction Synchronization Callback)

### 📌 名词定义

事务同步器模式是指**利用事务基础设施提供的钩子，将那些必须在数据库事务 Commit 成功后才能执行的“旁路操作（如发 MQ、清缓存）”，注册到当前活跃的数据库事务上下文中，实现“只有 Commit 成功，才触发旁路操作”的原子性保护。** [1.1.2]

### 🧠 第一性原理

双写一致性痛点。在物理世界里，数据库 Commit 和向网络发送消息（MQ）是两个完全独立的行为，无法组成原子事务。  
如果先发 MQ 再提交事务，可能发生“事务回滚，但消息已发出”的**幻影消息灾难**。  
**本质解法：以数据库物理 COMMIT 的结果作为发射 MQ 的唯一引线。** [1.1.2]

### 🛠️ 落地实现（Spring 声明式 @TransactionalEventListener）

code

```Java
@Component
public class CacheCleaner {

    @Async // 异步执行，不阻塞主线程
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT) // 默认值 [1.1.3]
    public void on(OrderPaidEvent event) {
        // 只有事务 Commit 成功后，这里才会被触发！彻底杜绝幻影消息 [1.1.2]
        redisTemplate.delete("order:" + event.orderId());
    }
}
```

### ❌ 生产级特大避坑红线

- **写丢失（Write Loss）大坑**：由于 AFTER_COMMIT 阶段主事务已经彻底 Commit 并处于收尾期，**在此监听器内部直接调用 repo.save() 修改数据库会静默失效（不报错但写不进去）** [1.2.5]。
    
- **解法**：若在此阶段必须写库，**必须**在监听器上加上 @Transactional(propagation = Propagation.REQUIRES_NEW) 强制开启全新物理事务。