这是一份非常高浓度的**资深架构师实战手册**。

这次对话跨度很大，从**代码细节**到**中间件原理**，再到**架构设计**和**工程效率**。为了让你能够随时查阅，我按照**技术领域**进行了重组和提炼，帮你构建一个结构化的知识图谱。

---

### 一、 Redis 与 缓存架构 (核心重点)

#### 1. 序列化选型 (Serialization)
*   **原则：** 可读性 vs 性能。
*   **80% 场景 (业务/DTO)：** **`GenericJackson2JsonRedisSerializer`**。配合 `ObjectMapper` 开启 `activateDefaultTyping`，存入 `@class` 信息，方便反序列化。
*   **20% 场景 (核心/大数据)：** **手动序列化** 或 **Kryo/Protobuf**。为了极致空间和性能，放弃可读性。
*   **Key 的处理：** 必须用 `StringRedisSerializer`。

#### 2. 缓存框架选型
*   **Spring Cache:** 适合简单的后台管理、低并发场景。功能较弱（TTL 配置麻烦、无防击穿）。
*   **JetCache (推荐):** 适合高并发 C 端。
    *   **优势：** 声明式 TTL、**二级缓存 (Local+Remote)**、自动刷新 (`@CacheRefresh`)、防击穿锁 (`@CachePenetrationProtect`)。
    *   **策略：** 建议全线拥抱 JetCache，替代 Spring Cache。

#### 3. 缓存一致性模式
*   **Write-Through / Write-Back:** **金融场景禁用**。代理写模式，要么慢，要么丢数据。
*   **Cache-Aside Pattern (旁路缓存/双写):** **业界标准**。
    *   **流程：** 更新 DB -> 删除 Cache。
    *   **延迟双删：** 并不是 Spring Cache 自带的。如果非要做，推荐用 **MQ 延迟消息**，不要用 `CompletableFuture` (不可靠)。
    *   **JetCache 方案：** 使用 `@CacheInvalidate` (删) 或者 `@CacheRefresh` (自动定时修补)，比双删更优雅。

---

### 二、 高并发与稳定性治理

#### 1. 限流算法与框架
*   **计数器：** 有临界点问题，少用。
*   **令牌桶 (Token Bucket):** 允许突发流量。适合 **API 网关、C 端接口**。框架：`Guava`, `Redisson`, `Sentinel WarmUp`。
*   **漏桶 (Leaky Bucket):** 强制匀速。适合 **MQ 消费端削峰**。框架：`Guava RateLimiter`, `Sentinel 排队等待`。
*   **框架推荐：** **Sentinel** (全能、可视化、热点参数限流)。分布式限流用 **Redisson**。

#### 2. 监控体系 (Observability)
*   **标准全家桶：**
    *   **数据展示：** Grafana
    *   **数据存储：** Prometheus
    *   **采集：** Exporter (Redis/MySQL) + Micrometer (JVM/JetCache)。**不要用 CacheCloud 这种侵入式的。**
*   **链路追踪：** **SkyWalking**。
    *   **原理：** Java Agent (静态插桩) + ByteBuddy 修改字节码。
    *   **DDD 监控：** 默认监控不到 Domain 层，需引入 `apm-toolkit-trace` 并加 `@Trace` 注解。

---

### 三、 DDD 领域驱动设计 (架构方法论)

#### 1. 战略设计 (Strategic)
*   **方法论：** **Event Storming (事件风暴)**。
*   **核心：** 识别领域事件 (Event) -> 划分上下文 (Bounded Context)。
*   **注意：** 此阶段**严禁**讨论入参 (Input)、载荷 (Payload) 等战术细节。
*   **工具：** **Obsidian + Excalidraw** (本地知识库 + 绘图)，或者 **Miro** (在线协作)。

#### 2. 战术设计 (Tactical)
*   **代码规范：**
    *   **Entity:** 严禁 `@Data`，使用 `@Getter` + `@ToString`，手写 `equals` (只比 ID)。
    *   **Application Service:** 也就是 Controller 下面那一层，可以依赖 Spring 注解 (`@Transactional`, `@Cacheable`)，作为胶水层。
    *   **Domain Service:** 保持纯净 (POJO)。若需缓存，定义 `Gateway` 接口，在 Infra 层实现 (DIP 依赖倒置)。

---

### 四、 工程效率与工具箱

#### 1. 线上诊断神器：Arthas
*   **原理：** Java Agent (动态 Attach) + Instrumentation API。
*   **四大神技：**
    *   `dashboard` + `thread -n 3`: 抓 **CPU 100%** / 死锁。
    *   `trace`: 抓 **接口慢** (定位耗时代码块)。
    *   `watch`: 抓 **参数错误** (无日志看入参返回值)。
    *   `redefine/retransform`: **线上热更** 代码救火。
*   **避坑：** 退出 Dashboard 按 `Q`。离开时用 `stop` 彻底卸载。

#### 2. 压测工具
*   **JMeter:** **核武器**。支持协议广 (Redis/JDBC/Dubbo)，并发高，适合做 SRE 级别的容量规划和一致性验证。
*   **Apifox:** **瑞士军刀**。适合日常开发、文档管理、功能测试。
*   **策略：** 开发用 Apifox，大促压测/中间件压测用 JMeter。

#### 3. 终端体验
*   **Windows Terminal:** 必须配置 **Git Bash** 或 **PowerShell 7** (配置 ListView 模式) 作为内核，否则 Arthas 体验很差。

---

### 五、 底层原理 (First Principles)

1.  **Git:** 本质是 **DAG (有向无环图)** + **快照 (Snapshot)**。`rebase` 是修剪历史，`merge` 是保留历史。
2.  **Redis Cluster:** **去中心化 (Gossip)**。为了极致性能和部署简单，没有 Controller/ZK，靠客户端 (Smart Client) 承担路由压力。
3.  **Java Agent:** JVM 的后门。SkyWalking 是启动时拦截 (`premain`)，Arthas 是运行时挂载 (`agentmain`)。

---

**给你的复习建议：**
你现在的状态是**“点状经验丰富，正在连点成线”**。
接下来的 2 周，建议你找一个简单的业务场景（如：用户积分系统），用 **Obsidian** 做一次事件风暴，用 **Spring Boot 3 + JetCache** 写代码，用 **Arthas** 调试，用 **JMeter** 压测。这套组合拳打下来，你的功力会完全恢复甚至超越以前。