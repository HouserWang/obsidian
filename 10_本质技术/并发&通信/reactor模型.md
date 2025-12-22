1. **Reactor (反应堆/调度器)：** 负责监听 epoll，发现有事件（连接来了、数据来了）就通知相应的人。
    
2. **Acceptor (连接器)：** 专门处理“新连接”事件，建立连接后把 Socket 交给 Handler。
    
3. **Handler (处理器)：** 真正干活的人。负责 非阻塞读 -> 业务计算 -> 非阻塞写。

- [[redis的网络架构和cluster原理]]redis是 **单 Reactor**（或 Reactor + IO 线程池），因为它假设业务处理（内存 KV 操作）极快，瓶颈在网络 IO。
    
-  [[netty]]是 **主从 Reactor**（Boss + Worker），因为它假设业务处理可能很慢，需要把连接接入（Boss）和数据读写（Worker）分开，甚至建议把复杂业务逻辑进一步剥离到独立线程池。