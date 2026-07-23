# 预留/票据模式 (Reservation / Ticket Pattern)

### 📌 名词定义

预留/票据模式是指**在微服务多库环境下，不使用重型分布式事务（如 Seata 2PC），而是通过将下游写操作降级为“预占/冻结（Try）”状态，并搭配带有 TTL（超时生存时间）的自动释放机制与最佳努力补偿，在业务层实现轻量级一致性的设计模式 [1.1.2]。**

### 🧠 第一性原理

与其使用分布式锁强行锁住多方数据库，不如**在业务上容忍“软状态（Soft State）”**。  
通过“锁定并不等于扣减，不确认就自动失效”的超时机制，在不需要任何重型一致性框架的前提下，达成极致的系统吞吐量。

### 🛠️ 落地实现（最佳努力补偿 + TTL）

codeJava

```
public void placeOrder() {
    // 1. 锁库存（库存服务内部设置 15 分钟 TTL 自动释放锁） [1.1.2]
    inventoryClient.lock(items, idempotencyKey); 

    try {
        // 2. 本地写单
        orderTransaction.createOrder(draft);
    } catch (Exception e) {
        // 3. 逆向补偿：本地失败，立刻最佳努力反向解锁 [1.1.2]
        try {
            inventoryClient.unlock(items, idempotencyKey); // [1.1.2]
        } catch (Exception ex) {
            // 解锁也失败不要紧，15分钟后库存服务的 TTL 自动释放该库存！ [1.1.2]
        }
        throw e;
    }
}
```

### ⚖️ 架构权衡

- **优点**：性能极佳，彻底避免了 Seata AT 等全局长事务锁带来的性能瓶颈 [3.2.1]；不依赖任何第三方分布式事务框架，可用性强。
    
- **缺点**：业务层必须设计“中间预留状态”（如锁定库存、冻结优惠券），且必须保证消费端的幂等性。