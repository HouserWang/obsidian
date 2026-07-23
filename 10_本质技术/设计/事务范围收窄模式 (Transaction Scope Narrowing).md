# 事务范围收窄模式 (Transaction Scope Narrowing)

### 📌 名词定义

事务范围收窄是指**将数据库事务（如 Spring 中的 @Transactional）的边界缩减到极致，仅包裹数据库的物理写操作，将所有只读查询、CPU 密集型计算和慢 I/O 远程调用（RPC/HTTP）彻底隔离在事务之外。**

### 🧠 第一性原理

数据库连接（Connection）是极其昂贵的、有限的临界资源。  

```
数据库连接占用时长=慢 I/O 耗时+计算耗时+物理 SQL 写入耗时数据库连接占用时长=慢 I/O 耗时+计算耗时+物理 SQL 写入耗时
```

  
如果将长链路方法直接加上 @Transactional，慢 I/O（如 RPC 抖动）会成倍延长连接占用时间，导致物理连接池瞬间枯竭。  
**本质解法：将事务持有时间压低至仅等于“物理 SQL 写入耗时”。**

### 🛠️ 落地实现（JPA/MyBatis 伪代码）

不要将 @Transactional 写在最外层 Service，而是提取一个专门的、极简的 Local Transaction 组件：



```java
// 1. 最外层应用服务：不开启事务，准备所有物料
public class OrderAppService {
    public void placeOrder() {
        queryCatalog(); // 事务外
        calculatePromotion(); // 事务外
        // 只有这里，才进入事务
        orderTransaction.writeDbInTinyTransaction(data); 
    }
}

// 2. 本地事务组件：短平快
@Component
public class OrderTransaction {
    @Transactional // 仅在这里开启事务
    public void writeDbInTinyTransaction(Data data) {
        insertOrder();
        updateInventory();
    }
}
```

### ⚖️ 架构权衡

- **优点**：极大提升数据库连接池吞吐量，彻底杜绝由于外部网络抖动引发的本地数据库连接池崩溃。
    
- **缺点**：增加了一个专门写 @Transactional 的内部 Service 组件，代码层级多了一层（增加了微小的代码复杂度）。