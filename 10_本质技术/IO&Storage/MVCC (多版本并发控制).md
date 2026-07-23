# MVCC (多版本并发控制)

## 📌 第一性原理

**读写不阻塞**：通过维护数据的多个历史版本，读操作读取旧版本，写操作创建新版本，两者互不阻塞，大幅提升并发性能。

---

## 🔍 核心概念

### 1. 多版本 (Multi-Version)
- 每次修改不直接覆盖原值，而是创建一个新版本
- 每个版本有唯一的时间戳或事务 ID
- 读操作根据隔离级别选择读取哪个版本

### 2. 快照读 (Snapshot Read)
- 读操作不获取锁，直接读取数据的历史版本
- 实现 **Repeatable Read (可重复读)** 隔离级别
- 例如：MySQL 的普通 SELECT 语句

### 3. 当前读 (Current Read)
- 读操作获取最新版本，并加锁
- 例如：MySQL 的 `SELECT ... FOR UPDATE`、`UPDATE`、`DELETE`

---

## 🗄️ MySQL InnoDB 实现

### Undo Log (回滚日志)
- 存储数据修改前的旧版本
- 用于事务回滚和 MVCC 读
- 每个版本通过 **回滚指针 (Rollback Pointer)** 链接成链表

### Read View (读视图)
- 判断当前事务能看到哪个版本的快照
- 包含：
  - `m_ids`：活跃事务 ID 列表
  - `min_trx_id`：最小活跃事务 ID
  - `max_trx_id`：下一个要分配的事务 ID
  - `creator_trx_id`：创建 Read View 的事务 ID

### 可见性判断规则
1. 若版本的 `trx_id` == `creator_trx_id` → 可见（自己修改的）
2. 若版本的 `trx_id` < `min_trx_id` → 可见（已提交）
3. 若版本的 `trx_id` >= `max_trx_id` → 不可见（未开始）
4. 若 `trx_id` 在 `m_ids` 中 → 不可见（仍活跃）
5. 否则，沿着 Undo Log 链表找下一个版本，重复判断

---

## 🎯 隔离级别与 MVCC

| 隔离级别 | MVCC 作用 | 实现方式 |
|---------|----------|---------|
| **Read Uncommitted (读未提交)** | ❌ 不使用 | 直接读最新版本，有脏读 |
| **Read Committed (读已提交)** | ✅ 使用 | 每次 SELECT 生成新的 Read View |
| **Repeatable Read (可重复读)** | ✅ 使用 | 事务开始时生成 Read View，之后复用 |
| **Serializable (串行化)** | ❌ 不使用 | 全表锁，完全串行 |

---

## 🔄 与其他技术的关系

- **Copy-on-Write (写时复制)**：MVCC 的底层实现常依赖写时复制
- **WAL (预写式日志)**：Undo Log 是 WAL 的一种，保证回滚能力
- **锁**：MVCC 减少锁竞争，但写操作仍需锁

---

## 💡 优势与劣势

| 优势 | 劣势 |
|-----|-----|
| 读写不阻塞，并发性能高 | 存储多个版本，占用空间 |
| 实现非阻塞读 | 需要后台清理旧版本（Purge） |
| 支持快照隔离 | 实现复杂 |

---

## 🔗 关联笔记

- [[mysql读写详细流程]] — MySQL InnoDB 中 MVCC 的实际应用
- [[分布式设计术语]] — Copy-on-Write、WAL 等相关术语
- [[写放大 vs 读放大]] — Undo Log 对写放大的影响
