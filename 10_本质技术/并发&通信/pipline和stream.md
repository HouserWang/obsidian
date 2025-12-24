#### . Pipeline (流水线)：一种“请求模型”

**本质词汇**：**并发请求 (In-flight Requests)**、**延迟隐藏 (Latency Hiding)**。

- **没有 Pipeline (Stop-and-Wait)**：
    
    - 你（Master）给 Follower 发一个数据包 A。
        
    - **停下来**，喝口茶，盯着屏幕等 Follower 回复 "ACK A"。
        
    - 收到 ACK 后，再发数据包 B。
        
    - **缺点**：大部分时间都在**空等**（Wait RTT）。带宽利用率极低。
        
- **有 Pipeline**：
    
    - 你（Master）给 Follower 发数据包 A。
        
    - **根本不等回复**，紧接着发数据包 B、C、D、E...
        
    - 过了一会儿，收到 "ACK A"，再过一会儿收到 "ACK B"...
        
    - **优点**：把网络像水管一样**填满**。虽然每个包到达的时间（延迟）没变，但单位时间内传的包多了（吞吐量巨大）。
        
    - **RocketMQ 的做法**：Master 会有一个发送窗口，允许几百个请求同时在网络上“飞”，而不必发一个等一个。
        

#### Stream (流)：一种“数据形态”

**本质词汇**：**无边界 (Unbounded)**、**字节序列 (Byte Sequence)**。

- **Packet (包/帧)**：
    
    - Raft 的标准做法。一个 RPC 请求就是一个完整的包。比如 AppendEntriesRequest，它是一个 Java 对象序列化后的独立数据块。
        
    - DLedger 就是这种，它还是有“包”的概念，每一条日志是一个 Entry。
        
- **Stream (流)**：
    
    - RocketMQ 4.x **默认 HA (Static)** 的做法。
        
    - Master 根本不在乎这一堆字节是一条消息还是半条消息。它直接把 CommitLog 文件里的字节，通过 FileChannel.transferTo 灌入 Socket。
        
    - **Slave 的视角**：我收到的不是“对象”，而是像水龙头流出来的**字节流**。Slave 一边接水，一边自己切分“哦，这 100 字节是一条消息，存下来”。
        

**一句话总结：DLedger 用的是 Pipeline（并发发包），而 RocketMQ 默认 HA 用的是 Stream（字节流直灌）。**