<%* 
  // 可选：强制焦点到编辑器（解决某些新建笔记焦点问题）
  app.workspace.activeLeaf.view.editor.focus();

  // 弹出输入标题
  let title = await tp.system.prompt("请输入技术沉淀标题（例如：Redis 缓存穿透原理）", "");
-%>
---
created: <% tp.file.creation_date("YYYY-MM-DD HH:mm") %>
tags:
  - brag-doc
  - knowledge-block
category: <% await tp.system.suggester(["中间件原理", "架构设计(DDD)", "性能优化", "线上排查", "工程效能"], ["Middleware", "Architecture", "Performance", "Troubleshooting", "Engineering"]) %>
tech_stack: <% await tp.system.multi_suggester(["Redis", "Kafka", "MySQL", "RocketMQ", "Elasticsearch", "Dubbo", "Spring Cloud", "Kubernetes", "其他"], ["Redis", "Kafka", "MySQL", "RocketMQ", "Elasticsearch", "Dubbo", "Spring Cloud", "Kubernetes", "其他"]) %>
status: 🟢 掌握
---
# 🚀 技术沉淀: <% title %>

[[CURSOR1]]  // 这里先输入核心标题描述或关键词

## 1. 第一性原理 (The "Why" & "How")
> 💡 **核心机制拆解**：跳出 API，深入 OS 内核、网络模型或算法本质。
- **底层机制**：(例如：PageCache, Zero-Copy, Gossip, HashSlot, 倒排索引...)
- **设计哲学**：(例如：AP vs CP, 读写分离, 空间换时间...)
- **关键细节**：
    - [[CURSOR2]]  // 这里填写关键细节

## 2. 横向对比 (The Trade-off)
> ⚖️ **架构师视角**：没有最好的技术，只有最适合的权衡。
| 维度       | 当前技术          | 对标技术 (e.g. Kafka/Redis/ZK) |
|------------|-------------------|-------------------------------|
| **一致性** | [[CURSOR3]]       |                               |
| **IO 模型** |                   |                               |
| **适用场景** |                 |                               |
- **核心差异点**：[[CURSOR4]]

## 3. B 端业务落地 (The Value)
> 💼 **场景化应用**：在非高并发场景下，这个技术能解决什么痛点？(数据一致性、可维护性、系统解耦)
- **痛点描述**：[[CURSOR5]]
- **解决方案**：
- **收益/价值**：

## 4. 简历话术预案 (Resume Snippet)
> 📝 **降维打击**：明年写简历时，直接复制这一段。
> *格式建议：掌握 [技术原理]，在 [业务场景] 中，通过 [技术手段]，解决了 [什么问题/提升了什么指标]。*
- **话术 1 (侧重原理)**：深入理解 **<% title %>** 原理，... [[CURSOR6]]
- **话术 2 (侧重实战)**：在 **...** 业务中，设计了 **...** 方案，解决了 **...** 问题。

---
## 🔗 关联知识
- [[相关概念1]]
- [[相关概念2]]

[[CURSOR7]]  // 最后光标停在这里，便于继续添加