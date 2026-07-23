# LSM-Tree (日志结构合并树)

## 📌 第一性原理

**用读放大换写放大**：将随机写转化为顺序写，通过后台 Compaction 合并文件，适合写多读少的场景。

---

## 🗂️ 核心结构

### 1. MemTable (内存表)
- 内存中的有序结构（通常是跳表或红黑树）
- 写入直接追加到 MemTable
- 满了后转为 Immutable MemTable，后台刷盘

### 2. SSTable (Sorted String Table)
- 磁盘上的有序不可变文件
- 按 Level 分层存储（Level 0 → Level 1 → ...）
- Level 0 可能有重叠，高层无重叠

### 3. WAL (预写日志)
- 写入 MemTable 前先写 WAL
- 防止宕机丢数据
- MemTable 刷盘后可删除

---

## 📝 写入流程

1. **写 WAL**：操作先记录到磁盘 WAL
2. **写 MemTable**：写入内存 MemTable
3. **MemTable 满**：转为 Immutable MemTable，创建新 MemTable
4. **后台刷盘**：Immutable MemTable 刷盘为 Level 0 SSTable
5. **Compaction**：后台合并 SSTable，清理删除/更新数据

---

## 🔍 读取流程

1. **查 MemTable**：先查当前 MemTable
2. **查 Immutable**：再查 Immutable MemTable
3. **查 Level 0**：遍历 Level 0 SSTable（可能重叠）
4. **查高层**：从 Level 1 开始，每层只需查一个 SSTable（无重叠）
5. **合并结果**：取最新版本的数据

---

## 🔄 Compaction (合并)

### 为什么需要 Compaction？
- 删除和更新在 LSM-Tree 中是追加标记，需要合并清理
- 减少读放大（文件越少，读的次数越少）
- 回收空间

### Compaction 策略

#### Tiered Compaction (分层合并)
- 每层多个文件，大小相近
- 下层满了，把整个层合并到下一层
- 写放大小，但读放大和空间放大较大
- 适用于 Cassandra、RocksDB 部分场景

#### Leveled Compaction (水平合并)
- Level 0 可重叠，高层无重叠
- 每层有固定大小阈值
- 选中一个文件，与下一层重叠文件合并
- 读放大小，但写放大较大
- 适用于 LevelDB、RocksDB 默认模式

---

## 📊 与 B+Tree 对比

| 维度 | LSM-Tree | B+Tree |
|-----|---------|--------|
| **写入性能** | 高（顺序写） | 低（随机写） |
| **读取性能** | 低（读放大） | 高（一次定位） |
| **范围查询** | 支持（SSTable 有序） | 支持（叶子节点链表） |
| **事务支持** | 较弱 | 强（MySQL InnoDB） |
| **典型应用** | HBase、RocksDB、LevelDB | MySQL、PostgreSQL |

---

## 💡 优势与劣势

| 优势 | 劣势 |
|-----|-----|
| 写入极快（顺序写） | 读放大（需查多层） |
| 易于水平扩展 | 空间放大（多版本 + 合并临时空间） |
| 适合写多读少 | Compaction 消耗 IO 和 CPU |
| 支持快照 | 实现复杂 |

---

## 🏗️ 实际应用

- **RocksDB**：Facebook 开源，Leveled Compaction，广泛用于存储引擎
- **LevelDB**：Google 开源，RocksDB 的前身
- **HBase**：基于 HDFS 的 LSM-Tree 数据库
- **Cassandra**：使用 Tiered Compaction
- **TiKV**：基于 RocksDB 的分布式 KV

---

## 🔗 关联笔记

- [[写放大 vs 读放大]] — LSM-Tree 是读放大换写放大的典型
- [[B+Tree (B+ 树)]] — 传统数据库的索引结构对比
- [[WAL (预写式日志)]] — LSM-Tree 的 WAL 机制
- [[分布式存储核心概念辨析手册]] — 存储核心概念对比
