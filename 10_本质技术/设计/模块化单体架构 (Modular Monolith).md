# 模块化单体架构 (Modular Monolith)

### 📌 名词定义

模块化单体是指**在同一个部署单元（单一 JVM 进程、单一 JAR 包）和单一物理数据库下，严格遵循限界上下文（Bounded Context）划分包结构，并使用单体内部通信（Spring Event/本地调用）代替 RPC（Dubbo/Feign）的物理形态 [3.1, 3.2]。**

### 🧠 第一性原理

康威定律（Conway's Law）的物理折中。许多团队为了划分职责盲目拆分微服务，结果承担了极高昂的“分布式税（网络延时、网络分区、部署成本）”。  
模块化单体主张：**“如果你们只有一个人、一个库，请把代码装进同一个物理进程运行，但用逻辑边界死锁，保留未来随时拆分微服务的能力 [3.2]。”**

### 🛠️ 落地实现（Spring Modulith）

codeText

```
com.example.shop
├── order (订单上下文)
│   ├── Order.java (聚合根)
│   └── OrderPaidEvent.java (公开领域事件)
│
└── inventory (库存上下文)
    └── internal (内部包 - Modulith 限制此包无法被 order 导入)
        └── InventoryListener.java (事件监听器 - 消费订单事件)
```

利用 ApplicationModules.of(ShopApplication.class).verify() 单元测试，强制在编译/打包期校验任何非法的跨上下文调用。

### ⚖️ 架构权衡

- **优点**：性能极高（零网络损耗 [2]）、部署极其简单、开发极其轻量；同时强制规范了开发人员的边界意识，未来向微服务拆分时的重构成本几乎为零。
    
- **缺点**：所有代码部署在一起，单点崩溃仍会波及全局。