---
tags: [mysql, 进阶, 事务, MVCC, undo, ReadView, 内核]
type: guide
created: 2026-10-13
updated: 2026-10-13
---

# 原理剖析 · undo 日志与 MVCC

> ⚠️ **本文是本知识库最重要的一篇。** MVCC 是 InnoDB 最核心的机制，也是面试必问。
> ⚠️ **必须配实验读**，见第五节。

## 学习目标

读完本篇，你应当能够：

1. 说清 undo log 记录了什么、为什么要有它
2. ⭐ **画出 undo 版本链**并解释回表读历史版本的过程
3. ⭐ **说出 ReadView 的四个字段和可见性判定算法**
4. 解释「RR 的快照读」为什么能实现可重复读
5. 解释「RC 和 RR 的 ReadView 差别」——⭐ **这是最容易考错的地方**
6. 解释 undo 表空间膨胀和 purge 的机制

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[mysql/01-入门阶段-基础教程/第04章-事务与权限/4.1-事务与ACID]] | 隔离级别和 MVCC 的概念 |
| [[mysql/03-进阶阶段-深度剖析/00-剖析方法论]] | 阅读方法 |

---

## 一、为什么需要 MVCC

### 问题

```sql
-- 场景：转账，A 扣钱 B 加钱
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

⚠️ **如果「读」也要加锁**：读操作必须等写操作完成，⚠️ **读写互相阻塞**，并发能力极差。

### MVCC 的思路

⭐ **核心思想：读不阻塞写，写不阻塞读。**

```
写操作：改数据，生成新版本（旧版本信息放 undo）
读操作：⭐ 找到「对当前事务可见的那个版本」，读它
```

⚠️ **读操作完全不加锁**（这是快照读）——⚠️ 它只读 undo 链上的历史版本。

📌 对照 PostgreSQL：⭐ **PG 也是 MVCC，但实现方式完全不同**（追加式 vs undo 式）。见 [[postgresql/03-进阶阶段-深度剖析/事务与MVCC内核/原理剖析-MVCC与快照隔离]]。

---

## 二、undo log 记录了什么

### 三类操作

| 操作 | undo 记录什么 | 回滚时 |
|------|-------------|--------|
| ⭐ **INSERT** | 「这条是新增的」（只记反向标记） | **删除这行** |
| ⭐ **DELETE** | 被删行的**完整原值** | **插回这行** |
| ⭐ **UPDATE** | 被改列的**旧值**（只记变化的列） | **改回旧值** |

⚠️ **⚠️ 关键机制**：⭐ **`undo log` 先写，然后才改数据页。** 这个顺序是崩溃恢复的前提——⚠️ 如果数据页改了但 undo 没写，回滚就找不到「原值」了。

### ⚠️ INSERT 的 undo 特别之处

```sql
-- 场景：A 事务 insert 了但没提交，B 事务读
INSERT INTO t (id, v) VALUES (1, 'x');    -- A 未提交
SELECT * FROM t WHERE id = 1;              -- B 读不到
```

⚠️ **因为 INSERT 的 undo 只记「删掉这行」的标记，不记完整数据。** ⚠️ **⚠️ 这意味着：未提交的新行对其他事务完全不可见**（不需要读到「它存在但看不到」的中间态）。

⭐ **📌 记忆锚点：INSERT 的 undo 最短，DELETE 的 undo 最长（要插回完整行）。**

### undo log 的两个作用

```
① 回滚（ROLLBACK）—— 按 undo 反向操作
② ⭐ MVCC 读历史版本 —— 顺着 undo 链找可见版本
```

⚠️ **⚠️ 第一个作用是「主动」的（用户请求回滚），第二个是「被动」的（读操作顺链找版本）。** 后者是 MVCC 的关键。

---

## 三、⭐ undo 版本链

### 机制

```
初始状态：
  行 id=1, balance=100
  这行的隐藏列 trx_id = 50（最后修改它的事务 ID）

事务 A（trx_id=100）在 10:00 改成 90：
  ① 先写 undo：[trx_id=100][before balance=100][roll_ptr → 指向上一个版本]
  ② 改数据页：balance=90, trx_id=100
  ③ roll_ptr 指向 undo 里的「上一版本」

事务 B（trx_id=200）在 10:01 改成 80：
  ① 先写 undo：[trx_id=200][before balance=90][roll_ptr → 指向 trx_id=100 的版本]
  ② 改数据页：balance=80, trx_id=200
```

**⭐ 形成的版本链**：

```
数据页（最新）：
┌────────────────────────────────────┐
│ id=1, balance=80, trx_id=200       │
│ roll_ptr ───────────────────────────────┐
└────────────────────────────────────┘    │
                                          ▼
undo log：
┌────────────────────────────────────┐
│ ① [trx_id=200][before=90]           │  ← roll_ptr 指到这里
│    roll_ptr ───────────────┐        │
└────────────────────────────┼────────┘
                             ▼
              ┌────────────────────────────────────┐
              │ ② [trx_id=100][before=100]         │
              │    roll_ptr ───────────┐           │
              └────────────────────────┼───────────┘
                                       ▼
                         ┌──────────────────────────┐
                         │ ③ 最初的版本（无 undo）  │
                         │    balance=100           │
                         └──────────────────────────┘
```

⭐ **读操作的本质**：⭐ **从最新版本出发，顺着 `roll_ptr` 往回走，找到「对自己可见」的那个版本。**

### 隐藏列

⚠️ **InnoDB 的每行都有三个隐藏列**（用户看不到）：

| 隐藏列 | 作用 |
|--------|------|
| ⭐ `DB_TRX_ID` | 最后修改这行的事务 ID |
| ⭐ `DB_ROLL_PTR` | ⭐ **指向 undo 版本链的指针** |
| `DB_ROW_ID` | ⚠️ **没有主键时用它**（6 字节，⚠️ 见 [[mysql/02-中段阶段-专题精讲/InnoDB存储引擎/专题概览]]） |

⚠️ **这三个隐藏列是 MVCC 能实现的基础。** ⚠️ 它们的代价：每行多 6~13 字节。

---

## 四、⭐ ReadView

### 四个字段（⚠️ 这是核心考点）

⚠️ **一个事务做「快照读」时，会创建一个 ReadView，存四个东西：**

| 字段 | 名字 | 含义 |
|------|------|------|
| ⭐ `m_ids` | 活跃事务列表 | ⭐ **创建 ReadView 时，所有还没提交的事务 ID 集合** |
| `min_trx_id` | 最小活跃事务 ID | `m_ids` 的最小值 |
| ⭐ `max_trx_id` | 下一个将要分配的事务 ID | ⚠️ **不是当前最大事务 ID，是「下一个」** |
| `creator_trx_id` | 创建者事务 ID | ⭐ **创建这个 ReadView 的事务自己的 ID** |

### ⚠️ `max_trx_id` 的坑

⚠️ **⚠️ `max_trx_id` 不是「当前最大事务 ID」，而是「下一个会被分配的事务 ID」。**

```
现有事务：50（已提交）、100（活跃中）
创建 ReadView 时，下一个事务 ID 是 101
  ↓
所以 max_trx_id = 101，而不是 100
  ↓
含义：「ID >= 101 的事务，一定是在我之后创建的，看不到」
```

⚠️ **⚠️ 这个设计是为了让 ID 全局单调递增**——⭐ 「比 `max_trx_id` 大的 ID，一定是未来才创建的事务」。

---

## 五、⭐⭐ 可见性判定算法

⚠️ **这是本篇最重要的部分。面试必问。**

⚠️ **⚠️ 关键前提：⚠️ 事务 ID 是「单调递增」的**（⚠️ **❌ 不是全局顺序，是按 `max_trx_id` 比较「是否在我之前创建」的近似判断**）。

### 判定一条数据是否可见

对**版本链上的一条记录**，它的 `DB_TRX_ID`（记作 `trx_id`），用 ReadView 的四个字段判定：

| # | 条件 | 结果 | 理由 |
|---|------|------|------|
| ⭐ **1** | `trx_id == creator_trx_id` | ⭐ **可见** | ⭐ **「自己改的当然看得见」** |
| ⭐ **2** | `trx_id < min_trx_id` | ⭐ **可见** | ⚠️ 「我创建之前就已经提交了」（在 `m_ids` 之外） |
| ⭐ **3** | `trx_id >= max_trx_id` | ❌ **不可见** | ⚠️ 「我创建之后才有的事务」 |
| ⭐ **4** | `min_trx_id <= trx_id < max_trx_id` | ⚠️ **看 `m_ids`** | |
| 4a | | ⚠️ `trx_id` **在 `m_ids` 里** → ❌ **不可见** | 「在我创建时它还没提交」 |
| 4b | | ⚠️ `trx_id` **不在 `m_ids` 里** → ⭐ **可见** | 「在我创建前它已提交」 |

⚠️ **⚠️ 不可见时的动作**：**顺 `roll_ptr` 往下一个版本，继续判定。** ⚠️ **直到找到可见版本，或者到版本链末尾（说明这条行是我插入后未提交 → 对我不可见 → 返回「不存在」）。**

### 伪代码

```python
def is_visible(trx_id, readview):
    if trx_id == readview.creator_trx_id:      # 条件 1
        return True
    if trx_id < readview.min_trx_id:          # 条件 2
        return True
    if trx_id >= readview.max_trx_id:         # 条件 3
        return False
    # 条件 4：在 [min, max) 区间
    if trx_id in readview.m_ids:              # 4a
        return False
    return True                                # 4b
```

⭐ **📌 记忆锚点：「自己可见；小的可见；大的不可见；中间的问活跃列表」。**

---

## 六、⚠️ RC vs RR：ReadView 的差别

⚠️ **⭐ 这是最容易考错的地方。**

### 差别只有一句话

| 隔离级别 | ReadView 创建时机 | 效果 |
|---------|----------------|------|
| ⭐ **RR**（可重复读） | ⭐ **事务里第一次「快照读」时创建，之后一直复用** | ⚠️ **整个事务看到同一份快照**（可重复读） |
| ⭐ **RC**（读已提交） | ⭐ **⚠️ 每一条 SELECT 都创建新的 ReadView** | ⚠️ **能看到别人新提交的数据** |

### 实验验证

```sql
-- 建表和数据
CREATE TABLE t (id INT PRIMARY KEY, v INT);
INSERT INTO t VALUES (1, 100);

-- 会话 A
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
SELECT * FROM t WHERE id = 1;      -- v=100，创建 ReadView

-- 会话 B（另一个连接）
UPDATE t SET v = 200 WHERE id = 1;
COMMIT;

-- 会话 A 继续
SELECT * FROM t WHERE id = 1;      -- ⭐ RR: 仍然 100（复用 ReadView）
COMMIT;
-- ⭐ RC: 200（新建了 ReadView）
```

⚠️ **⚠️ 这个实验是本篇最重要的验证，一定要跑。** ⚠️ 它直接证明了「RR 的可重复读是靠 ReadView 复用实现的」。

---

## 七、⚠️ 快照读 vs 当前读

⚠️ **本篇第二个容易错的点。**

### 两种读

| 类型 | 用什么 | 加锁吗 | 读到什么 |
|------|-------|--------|---------|
| ⭐ **快照读**（一致性非锁定读） | MVCC + ReadView | ⭐ **完全不加锁** | 快照时刻的版本 |
| ⭐ **当前读**（一致性锁定读） | ⭐ **最新版本 + 锁** | ✅ **加锁** | ⭐ **最新已提交版本** |

**哪些语句是哪种读**：

| 语句 | 类型 |
|------|------|
| `SELECT * FROM t WHERE id = 1` | ⭐ **快照读** |
| `SELECT * FROM t WHERE id = 1 LOCK IN SHARE MODE` | 当前读（共享锁） |
| ⭐ `SELECT * FROM t WHERE id = 1 FOR UPDATE` | ⭐ **当前读**（排他锁） |
| ⭐ `UPDATE t SET v = 2 WHERE id = 1` | ⭐ **当前读** |
| ⭐ `DELETE FROM t WHERE id = 1` | ⭐ **当前读** |

### ⚠️ 混用导致的诡异现象

```sql
-- RR 隔离级别
BEGIN;

SELECT * FROM t WHERE id = 1;      -- 快照读：v = 100
                                -- ⚠️ 注意：创建了 ReadView

UPDATE t SET v = 200 WHERE id = 1;  -- 当前读：更新（+ 加锁）

SELECT * FROM t WHERE id = 1;      -- ⭐ 快照读：v 仍然是 100！
                                -- ⚠️ 因为 ReadView 是 100 时刻的快照

SELECT v FROM t WHERE id = 1 FOR UPDATE;  -- ⭐ 当前读：v = 200

COMMIT;
-- 之后的新事务能看到 200
```

⚠️ **⚠️ 「我刚改成 200，读出来还是 100」——这是 MySQL 的正确行为，不是 bug。** ⚠️ 但如果你的业务逻辑依赖「改完立刻读到新值」，⚠️ **必须用 `FOR UPDATE`（当前读）**。

⭐ **📌 记忆锚点：RR 是「快照」语义，所以「快照读是旧数据」。想读最新就 FOR UPDATE。**

---

## 八、undo 表空间膨胀与 purge

### ⚠️ 为什么 undo 会膨胀

```
事务 A 在 10:00 创建 ReadView
    ↓
之后所有事务的每次修改都产生 undo
    ↓
⚠️ purge 线程想清理「不再被需要」的 undo
    ↓
⚠️ 但 A 的 ReadView 可能还要看 10:00 之后的数据
    ↓
⛔ 无法清理 → undo 无限堆积
```

### history list length

⚠️ **`history list length` = undo 链上「最老未清理的事务」到现在的 undo 条数。**

```sql
SHOW ENGINE INNODB STATUS\G
-- History list length 234567
```

⚠️ **⚠️ 这是长事务的终极指标**（详见 [[mysql/02-中段阶段-专题精讲/事务与并发控制/长事务与undo膨胀]]）。

### purge 线程的工作

```
① 从 history list 头部（最老的 undo）开始
② 检查有没有活跃事务的 ReadView 会用到它
   ├── 有 → ⛔ 停止（不能删）
   └── 没有 → 删除这条 undo，把空间标记为可复用
③ 继续下一条
```

⚠️ **⚠️ purge 是「从老到新」单向推进的**——⚠️ ⚠️ **一旦被最老的 ReadView 卡住，后面所有 undo 都清不掉。**

### 独立 undo 表空间（8.0）

⚠️ **MySQL 8.0 默认有独立的 undo 表空间**（`undo_001`、`undo_002`）：

| 优势 | 说明 |
|------|------|
| ⭐ **可在线扩容** | `ALTER TABLESPACE ... ADD DATAFILE`（⚠️ 5.7 不行） |
| 可独立备份/恢复 | |
| 写放大降低 | ⚠️ `innodb_undo_log_truncate` 默认 ON（⚠️ 5.7 默认 OFF） |

```sql
SHOW VARIABLES LIKE 'innodb_undo%';

-- 8.0 在线加 undo 文件
ALTER TABLESPACE innodb_undo_001 ADD DATAFILE 'undo_003' SIZE 256M;
```

⚠️ **⚠️ 但 undo 表空间过大本身是问题**——⚠️ **扩容只是应急，根治是解决长事务。**

---

## 九、⭐⭐ 实验清单

⚠️ **⚠️ 必须在隔离环境跑！用 Docker 起一个测试实例。**

### 实验 1：验证 undo 版本链的存在

```sql
-- 会话 A
BEGIN;
UPDATE accounts SET balance = 90 WHERE id = 1;
-- 不提交

-- 会话 B（另开连接）
SHOW ENGINE INNODB STATUS\G     -- 看 TRANSACTIONS 段落
-- ⚠️ 应该能看到事务 A 和它持有的锁

-- 会话 B 快照读
SELECT * FROM accounts WHERE id = 1;   -- 看到 100（快照读，MVCC）
```

### 实验 2：验证 RC vs RR 的 ReadView 差别 ⭐ 最重要

```sql
-- 建表
CREATE TABLE t (id INT PRIMARY KEY, v INT);
INSERT INTO t VALUES (1, 100);

-- 会话 A：RC
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
SELECT v FROM t WHERE id = 1;      -- 100

-- 会话 B
UPDATE t SET v = 200 WHERE id = 1;
COMMIT;

-- 会话 A 继续
SELECT v FROM t WHERE id = 1;      -- ⭐ RC: 200（新建 ReadView）
COMMIT;

-- ⭐ 再用 RR 重复一遍，这次会话 A 第二次读是 100
```

⭐ **⚠️ 这个实验做完，你就真正理解 RC 和 RR 的区别了。**

### 实验 3：验证快照读 vs 当前读

```sql
-- RR 隔离级别
BEGIN;
SELECT v FROM t WHERE id = 1;              -- 快照读：100
UPDATE t SET v = 300 WHERE id = 1;          -- 当前读：改
SELECT v FROM t WHERE id = 1;              -- ⭐ 快照读：还是 100
SELECT v FROM t WHERE id = 1 FOR UPDATE;   -- ⭐ 当前读：300
COMMIT;
```

### 实验 4：观察 undo 膨胀

```sql
-- 会话 A
BEGIN;
UPDATE big_table SET v = v + 1;     -- 批量改
-- 不提交

-- 另一个连接
SHOW ENGINE INNODB STATUS\G     -- 观察 History list length 增长

-- 会话 A
COMMIT;
-- 观察 History list length 逐步下降（purge 在工作）
```

⚠️ **这个实验能让你直观感受「长事务的代价」。**

### 实验 5：⭐ 对照 PG 的追加式 MVCC

```sql
-- 在 PG 上做实验 1 的等价操作
BEGIN;
UPDATE accounts SET balance = 90 WHERE id = 1;
-- 另一个连接看表大小
SELECT pg_size_pretty(pg_total_relation_size('accounts'));  -- ⭐ 变大了！

COMMIT;
VACUUM accounts;   -- ⭐ PG 需要 VACUUM 回收
SELECT pg_size_pretty(pg_total_relation_size('accounts'));
```

⭐ **⚠️ PG 的表会因为 UPDATE 而变大（追加式），MySQL 的表大小不变（undo 式）。** ⚠️ **这个对照实验能让你彻底理解两者的实现差异。**

---

## 十、⚠️ 陷阱

1. **❌ 以为 RR 的可重复读是靠「锁」实现的。** ⚠️ **是靠 ReadView 复用。** RR 的快照读**完全不加锁**。
2. **❌ 以为 `max_trx_id` 是「当前最大事务 ID」。** ⚠️ **是「下一个将要分配的 ID」。**
3. **❌ 记错可见性判定顺序。** ⚠️ **顺序很重要**：先看自己 → 再看小于 min → 再看大于等于 max → 最后查 `m_ids`。
4. **❌ 以为 RC 和 RR 的区别是「锁的粒度」。** ⚠️ **是「ReadView 的创建时机」。**
5. **❌ 在 RR 下混用快照读和当前读而不自知。** ⚠️ 「改完读还是旧值」。
6. **❌ 以为 UPDATE/DELETE 不用 MVCC。** ⚠️ 它们是「当前读」，加锁读最新。
7. **❌ 以为 purge 能清理所有 undo。** ⚠️ **被老 ReadView 卡住就清不掉。**
8. **❌ 认为 PG 和 MySQL 的 MVCC 是一回事。** ⚠️ 追加式 vs undo 式，行为差异很大。

---

## 小结

- **MVCC 的目标**：⭐ 读不阻塞写、写不阻塞读
- **undo log 的两个作用**：回滚（主动）+ MVCC 读历史版本（被动）
- **三个隐藏列**：`DB_TRX_ID`（最后修改的事务）、`DB_ROLL_PTR`（版本链指针）、`DB_ROW_ID`（无主键时用）
- ⭐ **undo 版本链**：⭐ 「每行有 `roll_ptr` 指向 undo；undo 里也有 `roll_ptr` 指向更早的版本；最终连到原始版本」
- ⭐ **ReadView 四字段**：`m_ids`（活跃事务集合）、`min_trx_id`、`max_trx_id`（⚠️ **是「下一个」不是「当前最大」**）、`creator_trx_id`
- ⭐ **可见性判定五步**：**自己→可见；`< min`→可见；`>= max`→不可见；`[min,max)` 区间查 `m_ids`（在→不可见，不在→可见）**
- ⭐ **RC vs RR**：⭐ **RR 第一次快照读创建 ReadView 并复用；RC 每条 SELECT 都新建**
- ⭐ **快照读（不加锁，ReadView）vs 当前读（加锁，最新版本）**；`UPDATE`/`DELETE`/`FOR UPDATE` 是当前读
- ⚠️ **RR 下混用两种读** → 「改完快照读还是旧值」
- **purge 从老到新单向推进**，⚠️ **被最老的 ReadView 卡住就全清不掉** → history list length 膨胀

## 练习

1. MVCC 要解决什么问题？它的核心思想是什么？
2. INSERT / DELETE / UPDATE 三种操作的 undo 分别记录什么？为什么 INSERT 的最简单？
3. 画出 undo 版本链的结构。一行数据里有哪三个隐藏列？
4. ⭐ 完整说出 ReadView 的四个字段。其中 `max_trx_id` 到底是什么？为什么这样设计？
5. ⭐ **背出可见性判定的五个步骤和判定表。**
6. RC 和 RR 的唯一区别是什么？用一个实验证明。
7. 什么是快照读和当前读？哪些语句属于当前读？
8. 为什么 RR 下「改了数据再 SELECT，读到的是旧值」？怎么读到新值？
9. purge 线程为什么会被「卡住」？history list length 涨到 100 万意味着什么？
10. MySQL 和 PostgreSQL 的 MVCC 实现有什么不同？会导致什么现象？

## 导航

[[mysql/03-进阶阶段-深度剖析/00-剖析方法论|剖析方法论]] · [[mysql/02-中段阶段-专题精讲/事务与并发控制/长事务与undo膨胀|长事务与undo膨胀]] · 下一篇：[[mysql/03-进阶阶段-深度剖析/事务与锁内核/原理剖析-锁体系与死锁]] · 对照：[[postgresql/03-进阶阶段-深度剖析/事务与MVCC内核/原理剖析-MVCC与快照隔离|PG 的 MVCC]]
