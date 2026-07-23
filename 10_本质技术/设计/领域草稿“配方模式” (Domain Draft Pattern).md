# 领域草稿/配方模式 (Domain Draft Pattern)

### 📌 名词定义

领域草稿模式是指**在创建复杂的聚合根（Aggregate Root）之前，在事务外部将所有的关联数据、远程快照和规则计算结果整合成一个无状态的“草稿（Draft）”或“配方（Recipe）”对象，再将此草稿一次性传入聚合根的创建工厂中。**

### 🧠 第一性原理

根据 DDD 规范，聚合根在创建出来的那一瞬间，就必须是**完整且合法的（Create-Is-Valid 约束）**。  
然而，创建聚合根往往需要依赖大量的外部上下文（如查库验证、查价格、查促销）。为了不让聚合根在初始化时直接调用 Repository 或外部 Service 造成耦合，我们先在外部拼装好一份完整的“原料清单”（草稿），然后再在内存中构建干净的聚合根。

### 🛠️ 落地实现


```Java
// 1. 业务草稿：承载所有外部依赖计算完后的快照
public record PlaceOrderDraft(
    Command cmd, StoreSnapshot store, PriceSnapshot price, InventorySnapshot inv
) {}

// 2. 聚合根：只认草稿，不认外部 RPC 客户端
public class Order {
    public static Order create(PlaceOrderDraft draft) {
        // 在内存中无感校验，创建合法的聚合根
        validate(draft);
        return new Order(draft);
    }
}
```

### ⚖️ 架构权衡

- **优点**：保持了聚合根（Entity）的绝对纯净性，使其不产生任何 I/O 依赖；同时让复杂的应用服务（AppService）多了一层强类型的阶段性产物，极利于单元测试。
    
- **缺点**：多写了一个 Draft 记录类。