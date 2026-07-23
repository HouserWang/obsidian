# 本地消息表 / 信箱模式 (Transactional Outbox Pattern)

### 📌 名词定义

本地消息表模式是指**在业务数据库中建立一张额外的 outbox_event（信箱）表。当发生状态改变时，业务数据更新和事件记录在一笔本地数据库事务（JTA/ACID）内写入。随后由后台的异步线程（轮询、CDC 或事件驱动）将事件从信箱拉取并安全送达 MQ [1.1.2, 1.1.7]。**

### 🧠 第一性原理

解决分布式系统中的 **“至少送达一次（At-Least-Once）”** 投递问题。它解耦了“业务落盘”与“网络投递”，确保哪怕 MQ 发生 1 小时断网，本地事件也永远不会丢失，网络恢复后能自动重试，达成数据的**最终一致性**。

### 🛠️ 落地实现（大厂标准 CDC 架构）

codeText

```
[本地事务] ──> 写入数据库 (Order表 + outbox_event表)
                     │
                     v (写 Binlog 日志)
               [数据库 Binlog]
                     │
                     v (非阻塞、毫秒级解析日志)
               [Debezium / Canal] (CDC 监听工具)
                     │
                     v 
               [Kafka / RabbitMQ] ──> [最终落盘/标记完成]
```

### ⚖️ 架构权衡

- **优点**：分布式数据一致性的行业标准，高并发、非阻塞、零延迟（配合 CDC 方案时），彻底保障数据绝不丢失。
    
- **缺点**：链路变长，引入了 CDC 抽取中间件，运维和监控成本上升。
    
- **轻量级妥协方案**：如果是 Spring 生态且项目规模中等，首选引入 **Spring Modulith**，它会自动帮你建表、拦截本地事件、持久化并安全派发，开发侵入几乎为零 [3.1, 3.3]。



### Outbox + 轮询发布

```java
	@Transactional
	
	public Order createOrder(PlaceOrderDraft draft) {
	
	    Order order = orderFactory.create(draft);
	
	    // 业务数据先落库。
	
	    orderRepository.save(order);
	
	    // 待发布事件也落库，和订单在同一个本地事务里。
	
	    outboxRepository.saveAll(order.pullDomainEvents());

	    return order;
	
	}
	
	@Scheduled(fixedDelay = 1000)

public void publish() {

    // 批量拉取待发布事件，避免一次扫太多。

    List<OutboxEvent> events = outboxRepository.findPending(100);

    for (OutboxEvent event : events) {

        try {

            // 事务提交之后，发布器再发送 MQ。

            messageTemplate.send(event.topic(), event.payload());

            // 发送成功后标记状态，下一轮就不会再扫到。

            outboxRepository.markPublished(event.id());

        } catch (Exception ex) {

            // 发送失败不丢，标记失败，后面继续重试。

            outboxRepository.markFailed(event.id(), ex.getMessage());

        }

    }

}
```
	
Outbox 后面接 MQ，还是接同步调用

Outbox 解决的是“后续动作不能丢”。

至于后续动作怎么执行，有两种常见形式。

第一种是 Outbox + MQ：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/Nzw1nr3Oqq5BekkwHauuXKDThicPDzuOBZnj1B0ibKOfv75ySOTz9hbaiaqLvpTro5vYcgJ1iaExfuYLLjtpn0Kiba80a21dJibCrb5ZKBsh40wfg/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&watermark=1#imgIndex=6)

它适合一个事件有多个消费者的场景，比如通知、履约、会员、统计。

第二种是 Outbox + 同步调用：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/Nzw1nr3Oqq7vMmoFKQ2E1yCL9cAmdkzpeWYPWribpeIDDAC8x8RozkMibfZv5B6o29sEz7nhaGAZC7z2xpzYa7qSxd2SuZdg1T675AAGwTibxQ/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&watermark=1#imgIndex=7)

它适合下游没有 MQ，只提供 HTTP/RPC 接口的场景。

但要注意，Outbox + 同步调用 更像可靠任务表。 如果以后消费者会越来越多，还是 Outbox + MQ 更合适。