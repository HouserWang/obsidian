´这是一篇**骨灰级**的 Redis 源码与架构深度复盘。

我们将把镜头拉得非常近，从**内核层面的 TCP 三次握手**开始，一直追踪到**磁盘 PageCache 的落盘**，完整还原一个 Redis Cluster 写请求的生命周期。

我们假设环境为 **Redis 6.0+ (开启多线程 IO)**，持久化配置为 **AOF (everysec)**。

---

### 总结：全链路数据流向图

```text
[Client]
   | (TCP SYN)
   v
[Kernel Accept Queue]
   | (epoll 触发 AE_READABLE)
   v
[Redis Main Thread] -> accept() -> 注册新 FD 到 epoll
   |
   v
[IO Threads] -> read() -> [User Input Buffer] -> 解析 RESP
   |
   v
[Redis Main Thread] -> 执行 SET -> [Redis Memory (Dict)]
   |
   +--> [AOF Buffer (User Space)]
   |     | (EventLoop 结束前)
   |     v
   |    write() 系统调用
   |     |
   |     v
   |    [Kernel PageCache]  <-- 此时断电会丢
   |     | (BIO 线程每秒)
   |     v
   |    fsync()
   |     |
   |     v
   |    [Physical Disk]     <-- 安全
   |
   +--> [Replication Buffer] -> write(slave_fd) -> [Slave]
   |
   v
[IO Threads] -> write(client_fd) -> [Kernel Socket Buffer] -> [Client]
```

### 第一阶段：连接建立 (Connection Phase)
**核心动作：内核握手 -> accept 系统调用 -> 注册 epoll -> 状态流转**

这是你要求的“连接事件转读写事件”的精确过程。

1.  **内核握手 (Kernel Handshake):**
    *   客户端发起 `SYN`。
    *   Redis Server 的 OS 内核处理 TCP 三次握手，将建立好的连接放入 **全连接队列 (Accept Queue)**。
    *   *此时 Redis 用户态进程还不知道有连接来了。*

2.  **触发 Accept 事件:**
    *   Redis 主线程的 EventLoop 正阻塞在 `epoll_wait` 上。
    *   因为 Redis 启动时监听了 Server Port (6379)，当全连接队列有数据，`epoll` 返回 **Listening Socket** 可读事件。

3.  **主线程接入 (Main Thread Accept):**
    *   主线程调用回调函数 `acceptTcpHandler`。
    *   执行 `accept()` 系统调用，从内核队列拿到一个新的 **Socket FD (文件描述符)**，代表这个客户端连接（例如 FD=10）。
    *   将 FD 设置为 **非阻塞 (Non-blocking)**。

4.  **事件注册 (Event Registration - 关键转换):**
    *   Redis 创建一个 `client` 结构体绑定这个 FD。
    *   **关键动作：** 调用 `aeCreateFileEvent(FD, AE_READABLE, readQueryFromClient)`。
    *   **底层操作：** 调用 `epoll_ctl(EPOLL_CTL_ADD)`，将这个新 FD 注册进 epoll，监听 **READ (可读)** 事件。
    *   *至此，连接建立完成，从“监听连接”转换为“监听该连接的数据读取”。*

---

### 第二阶段：写请求处理 (Write Request Processing)
**核心动作：epoll 触发 -> IO 线程读取 -> 主线程执行 -> 写入 AOF Buffer**

假设客户端发送 `SET user:1001 zhangsan`。

1.  **IO 多路复用触发:**
    *   数据包到达网卡 -> DMA 拷入内核 Buffer -> TCP 协议栈处理。
    *   `epoll_wait` 唤醒主线程，告知 FD=10 有数据可读。

2.  **IO 线程并行读取 (Redis 6.0+):**
    *   主线程并不直接读，而是将 Client 分配给 **IO Thread**（Round Robin 轮询分配）。
    *   主线程进入忙轮询（Busy Loop）等待 IO 线程完成。
    *   **IO Thread 动作：**
        *   调用 `read(FD)` 系统调用，将数据从 **内核 Buffer** 拷贝到 **用户态 Input Buffer**。
        *   解析 RESP 协议，将字节流转换为 Redis 命令对象（argv/argc）。

3.  **主线程串行执行 (Main Thread Execution):**
    *   IO 线程读完后，主线程接管。
    *   **Cluster 路由检查：** 计算 `CRC16("user:1001")`，判断 Slot 归属。如果不归我管，返回 `-MOVED`。
    *   **执行命令：** 调用 `setCommand`，修改内存中的 `redisDb`（HashTable）。
    *   *此时内存数据已更新。*

---

### 第三阶段：持久化与 PageCache (Persistence Phase)
**核心动作：User Buffer -> Kernel PageCache -> Disk**

这是你要求的“精确到 PageCache”的过程。

1.  **写入 AOF 缓冲区 (User Space):**
    *   命令执行成功后，Redis 将命令追加到 **`server.aof_buf`**。
    *   **注意：** 这只是一个 C 语言层面的 `sds` 字符串拼接，完全在 **Redis 进程堆内存** 中。

2.  **刷入 PageCache (Kernel Space):**
    *   在本次 EventLoop 循环即将结束前，Redis 调用 `flushAppendOnlyFile`。
    *   **系统调用：** `write(aof_fd, aof_buf, length)`。
    *   **数据流向：** 数据从 **Redis 用户态内存** 拷贝到了 **Linux 内核的 PageCache (页缓存)**。
    *   **耗时：** 极快（微秒级），因为不涉及磁盘 IO。
    *   *此时：如果 Redis 进程 Crash，数据不丢（在 OS 里）；如果服务器断电，数据丢失。*

3.  **落盘 (Disk Sync):**
    *   因为配置是 `appendfsync everysec`。
    *   **BIO 线程：** Redis 有一个后台线程 `bio_fsync`。
    *   它每秒钟对 `aof_fd` 执行一次 **`fsync()`** 系统调用。
    *   **内核动作：** 强制将 PageCache 中的脏页（Dirty Pages）调度写入物理磁盘。
    *   *此时：数据才算真正安全。*

---

### 第四阶段：数据复制 (Replication Phase)
**核心动作：Replication Buffer -> Socket Send**

1.  **写入复制积压缓冲区:**
    *   主线程把命令同时也写入 `server.repl_backlog` 和所有 Slave 的 `client->buf`。

2.  **异步发送:**
    *   在 EventLoop 的 `beforeSleep` 阶段，或者 IO 线程写回阶段。
    *   调用 `write(slave_fd)`，把数据发给 Slave。

---

### 第五阶段：响应返回 (Response Phase)
**核心动作：Output Buffer -> IO 线程写入 -> 网卡**

1.  **准备响应:**
    *   主线程执行完 SET 后，将 `+OK` 写入该 Client 的 **Output Buffer**（用户态内存）。

2.  **注册写事件 (或直接写):**
    *   Redis 6.0+ 会把 Client 再次放入队列，交给 **IO Thread**。

3.  **IO 线程并行写入:**
    *   IO Thread 调用 `write(client_fd)`。
    *   数据从 **用户态 Output Buffer** 拷贝到 **内核 Socket Send Buffer**。
    *   内核 TCP 栈负责将数据通过网卡发回给客户端。

---



这就是一个 Redis Cluster 写操作，从连接建立到数据落盘的**微观物理过程**。



|     |     |     |
| --- | --- | --- |
|     |     |     |
