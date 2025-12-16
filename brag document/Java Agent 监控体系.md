---
created: <% tp.file.creation_date("YYYY-MM-DD HH:mm") %>
tags:
  - brag-doc
  - knowledge-block
  - observability
  - skywalking
  - prometheus
category: 线上排查
tech_stack: [Java Agent, ByteBuddy, SkyWalking, Prometheus, Grafana, Micrometer]
status: 🟢 掌握
---

# 🚀 技术沉淀: Java Agent 监控体系：SkyWalking 链路追踪 vs Prometheus 指标监控

## 1. 第一性原理 (The "Why" & "How")
> 💡 **核心机制拆解**：跳出 API，深入 OS 内核、网络模型或算法本质。

- **底层机制**：
    1. **SkyWalking (Tracing)**：基于 **Java Agent (`premain`)** + **ByteBuddy** 字节码增强。在类加载时修改字节码，植入拦截器，利用 `ThreadLocal` 传递 TraceID，并通过 HTTP Header 实现跨进程透传。
    2. **Prometheus 体系 (Metrics)**：基于 **Pull 模型** + **TSDB (时序数据库)**。Java 端通过 **Micrometer** (门面模式) 暴露 `/actuator/prometheus` 端点，Prometheus 定时刮取数据。
- **设计哲学**：
    - **SkyWalking**：关注 **“微观的因果关系”** (Who called whom?)。数据量大，侧重采样率。
    - **Prometheus**：关注 **“宏观的趋势聚合”** (How is the system health?)。数据经过聚合，侧重数值变化。
- **关键细节**：
    - SkyWalking 默认监控不到 DDD 领域层，需通过 `apm-toolkit-trace` + `@Trace` 注解手动打点。
    - Prometheus 监控业务指标（如“下单失败数”）需要代码埋点 `Counter.increment()`，监控 JVM/中间件则依赖 `Exporters`。

## 2. 横向对比 (The Trade-off)
> ⚖️ **架构师视角**：没有最好的技术，只有最适合的权衡。

| 维度 | SkyWalking (APM) | Prometheus + Grafana (Metrics) |
| :--- | :--- | :--- |
| **核心数据** | **Trace (调用链)** + Log | **Metric (数值指标)** |
| **数据量级** | 极大 (全量采集会爆磁盘) | 较小 (只存聚合后的数值) |
| **排查场景** | **"为什么慢？"** (定位具体代码行/SQL) | **"是不是挂了？"** (报警、容量规划) |
| **侵入性** | **零侵入** (Agent 自动挂载) | **低侵入** (引入 Micrometer 包) |
| **存储成本** | 高 (需 ES/HBase 集群) | 低 (TSDB 压缩率高) |

- **核心差异点**：SkyWalking 是**“显微镜”**，用来查具体的病灶（哪个 SQL 慢）；Prometheus 是**“体温计”**，用来监控整体健康度（CPU 飙高、QPS 突增）。两者互补，不可替代。

## 3. B 端业务落地 (The Value)
> 💼 **场景化应用**：在非高并发场景下，这个技术能解决什么痛点？(数据一致性、可维护性、系统解耦)

- **痛点描述**：B 端业务逻辑复杂（DDD 分层多），出现 Bug 时日志分散，难以定位是 Domain 层逻辑错误还是底层 SQL 问题；同时缺乏对业务关键指标（如“支付成功率”）的实时大盘监控。
- **解决方案**：
    1. **链路可视化**：部署 SkyWalking，并针对 DDD 的 **Application Service** 和核心 **Domain Service** 添加 `@Trace` 注解，实现业务逻辑的全链路透视。
    2. **业务大盘**：引入 Micrometer，在关键业务节点（如支付回调）埋点 `Counter` 和 `Timer`，通过 Grafana 展示业务健康度。
- **收益/价值**：将跨服务/跨层级的故障定位时间（MTTR）从**小时级缩短至分钟级**；通过 Grafana 大盘提前发现业务异常（如支付接口成功率下跌），实现被动运维向主动治理的转变。

## 4. 简历话术预案 (Resume Snippet)
> 📝 **降维打击**：明年写简历时，直接复制这一段。
> *格式建议：掌握 [技术原理]，在 [业务场景] 中，通过 [技术手段]，解决了 [什么问题/提升了什么指标]。*

- **话术 1 (侧重原理)**：深入理解 **Java Agent** 字节码增强原理，掌握 **SkyWalking** 插件机制与 **ByteBuddy** 核心技术；构建了基于 **Tracing (SkyWalking)** + **Metrics (Prometheus)** + **Logging (ELK)** 的立体化可观测性体系。
- **话术 2 (侧重实战)**：针对 B 端复杂链路，通过 **SkyWalking 自定义埋点 (@Trace)** 解决 DDD 领域层监控盲区问题；结合 **Micrometer** 搭建业务指标监控大盘，实现了对核心业务链路的**全景可视化**与**秒级故障告警**。

---
## 🔗 关联知识
- [[Java_Agent_Bytecode]] (字节码增强原理)
- [[DDD_Tactical_Design]] (DDD 战术设计)
- [[Redis_Cluster_Monitor]] (中间件监控)