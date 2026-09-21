---
tags: [concept, PostgreSQL, 进阶, SSI, 可串行化, 写偏斜, 谓词锁, 序列化失败]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# 原理剖析 · SSI 可串行化隔离

## ① 现象

- `SERIALIZABLE` 下频繁报 `could not serialize access due to read/write dependencies`
- 同样的代码,`REPEATABLE READ` 不报错但数据逻辑出错
- 只读事务也会导致串行化失败
- 生产上"偶发的金额错算"难以复现

## ② 机制

### 快照隔离(SI)的能力边界

回顾 [[postgresql/03-进阶阶段-深度剖析/事务与MVCC内核/原理剖析-MVCC与快照隔离]]:

| 异常 | `READ COMMITTED` | `REPEATABLE READ`(= SI) | `SERIALIZABLE`(= SSI) |
|------|------------------|--------------------------|----------------------|
| 脏读 | ❌ | ❌ | ❌ |
| 不可重复读 | ✅ 可能 | ❌ | ❌ |
| 幻读 | ✅ 可能 | ❌ | ❌ |
| 丢失更新 | 靠行锁避免 | ❌(报错) | ❌(报错) |
| **写偏斜** | ✅ 可能 | ✅ **可能** | ❌ |
| **只读事务异常** | ✅ 可能 | ✅ **可能** | ❌ |

**SI 的漏洞:写偏斜(write skew)**

```
约束:会议预订系统中,同一时段至少保留 1 个空位

初始:slot A = 已订,slot B = 已订,空位数 = 0? 换一个更清晰的例子:

约束:每班至少 1 名医生在岗
  当前:Dr.X 与 Dr.Y 都在岗(count = 2)

事务 A:  SELECT count(*) FROM on_call WHERE shift='A' AND on_duty;  → 2
         （A 认为:我离岗后还有 1 人,安全）
事务 B:  SELECT count(*) FROM on_call WHERE shift='A' AND on_duty;  → 2
         （B 同样认为安全）
事务 A:  UPDATE on_call SET on_duty=false WHERE doctor='Dr.X';      提交 ✓
事务 B:  UPDATE on_call SET on_duty=false WHERE doctor='Dr.Y';      提交 ✓

结果:0 人在岗 —— 约束被违反
```

**为什么 SI 抓不到?**

- A 与 B **读的是不同行**(X 与 Y)
- A 与 B **写的是不同行**(X 与 Y)
- 无行级冲突 → 行锁不生效
- 各自的快照都"正确" → 快照隔离无从判断

**根本原因**:SI 只保证"每个事务看到的快照一致",不保证"并发结果等价于某个串行顺序"。

### SSI:可串行化快照隔离

`SERIALIZABLE` = SI + **依赖检测**。核心是跟踪事务之间的**读写依赖**。

### 三种依赖关系

| 依赖 | 记法 | 含义 |
|------|------|------|
| **写读依赖**(wr) | T1 →wr T2 | T1 写了 X,T2 读了 T1 写的 X(由此产生先后序) |
| **写写依赖**(ww) | T1 →ww T2 | T1 写 X 后 T2 又写 X(顺序由提交/加锁决定) |
| **读写反依赖**(rw) | T1 →rw T2 | T1 读了 X 的旧版本,T2 后来写了 X —— **这是"危险的"** |

**关键结论**:若依赖图出现**环**,则执行结果不等价于任何串行执行 → 必须终止一个事务。

**写偏斜在依赖图上表现为**:

```
A →rw B   (A 读了 Y 的旧值,B 后来写了 Y)
B →rw A   (B 读了 X 的旧值,A 后来写了 X)

→ 形成环:A → B → A  →  可串行化失败
```

> **两个 rw 反依赖构成的环,是写偏斜的本质特征。** SSI 检测的就是这种环。

### 谓词锁(SIREAD lock)

**问题**:读写反依赖涉及"**读取了一个范围**",而不仅仅是具体行。比如:

```sql
SELECT count(*) FROM on_call WHERE shift = 'A' AND on_duty;
```

这个查询"读"的是"`shift='A' AND on_duty` 这个谓词覆盖的行集合"。B 的 `UPDATE` 改变了这个集合的成员 —— 这是一种"范围级的读写冲突"。

**解法:SIREAD 锁(predicate lock / 谓词锁)**

| 特性 | 说明 |
|------|------|
| 粒度 | 可加在**元组、页、索引项、whole-relation** 上 |
| 是否阻塞 | **不阻塞任何事务**(只记录,不加锁等待) |
| 生命周期 | 通常持续到事务结束 |
| 存放 | 共享内存(受 `max_pred_locks_per_transaction` 等限制) |
| 内存优化 | 自动把"多个细粒度 SIREAD 锁"合并为"粗粒度"(如上提到 relation 级) |

```sql
SHOW max_pred_locks_per_transaction;   -- 默认 64
SHOW max_pred_locks_per_relation;
SHOW max_pred_locks_per_page;
```

> **SIREAD 锁不阻塞**,这是 SSI 能做到"可串行化但不牺牲并发"的关键。代价是共享内存与检测开销。

### 冲突与终止

```
检测时机:
  ① 事务提交时
  ② 发现新的依赖边时(主动检测)

检测逻辑:
  检查是否存在两个连续的 rw 反依赖(即 T1 →rw T2 →rw T1 这类环)
  若存在 → 选择代价较小的一方终止:
     ERROR: could not serialize access due to read/write dependencies among transactions
     SQLSTATE: 40001
     HINT: The transaction might succeed if retried.
```

**应用必须做两件事**:

1. **捕获 SQLSTATE `40001`** 并重试整个事务。
2. **把事务设计成可重试**(幂等、无外部副作用)。

```python
# 应用伪代码(必须包住整个事务)
import psycopg
from psycopg import errors

for attempt in range(5):
    try:
        with conn.transaction():          # isolation_level='SERIALIZABLE'
            ...  # 全部业务逻辑(不含外部调用)
        break
    except errors.SerializationFailure:   # 40001
        if attempt == 4: raise
        time.sleep(0.05 * (2 ** attempt))  # 指数退避
        continue
```

**最常见的错误做法**:只重试出错的那一条 SQL。**必须重试整个事务**,因为失败意味着"整个事务的读视图已不可信"。

### 只读事务异常

`SERIALIZABLE` 下,只读事务也可能安全失败:

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
-- DEFERRABLE 让只读事务等待一个"安全快照"再开始,
-- 从而保证它永远不会因串行化冲突而失败
```

| 选项 | 作用 |
|------|------|
| `READ ONLY` | 声明只读 |
| `DEFERRABLE` | 等待安全快照,从而**不会失败**(但会等待) |
| 两者组合 | **只读长报表的理想选择**:不失败 + 一致视图 |

> `SERIALIZABLE READ ONLY DEFERRABLE` 只对只读事务有效。写事务不可 `DEFERRABLE`。这是"长报表在 SERIALIZABLE 下不失败"的正确用法。

### 何时该用 SERIALIZABLE

| 场景 | 建议隔离级别 |
|------|--------------|
| 普通 CRUD | `READ COMMITTED`(默认) |
| 需要稳定读视图 | `REPEATABLE READ` |
| 报表只读 | `REPEATABLE READ`,或 `SERIALIZABLE READ ONLY DEFERRABLE` |
| **涉及"读后写"的约束检查** | `SERIALIZABLE` |
| 写偏斜风险的业务(排班、配额、拍卖) | `SERIALIZABLE` |

**判断标准**:若业务逻辑是"先查询一个聚合/集合,再基于它做写入",而这个**集合的成员可能被其他事务改变**,就需要 `SERIALIZABLE`。

**也可以用显式锁替代**

```sql
-- 方案 A:SERIALIZABLE(推荐,出错重试)
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- 业务逻辑
COMMIT;

-- 方案 B:显式物化冲突(乐观)
BEGIN;
SELECT count(*) FROM on_call WHERE shift='A' AND on_duty FOR UPDATE;   -- 锁住相关行
-- 但 FOR UPDATE 只能锁已存在的行,无法阻止"幻影插入"
COMMIT;

-- 方案 C:表级/谓词锁(悲观,牺牲并发)
BEGIN;
LOCK TABLE on_call IN SHARE ROW EXCLUSIVE MODE;
-- 业务逻辑
COMMIT;
```

| 方案 | 并发性 | 复杂度 | 正确性 |
|------|--------|--------|--------|
| `SERIALIZABLE` | 高 | 需重试逻辑 | ✅ 完整 |
| `FOR UPDATE` | 中 | 简单 | ⚠️ 无法防幻读 |
| 表锁 | 低 | 简单 | ✅ |

> **实践倾向**:能用 `SERIALIZABLE` + 重试解决就优先它;只有在重试代价不可接受(如事务很长)时才用显式锁。

### 性能代价

| 代价 | 说明 |
|------|------|
| SIREAD 锁内存 | 读操作越多,谓词锁越多,受 `max_pred_locks_*` 限制 |
| 检测开销 | 每次依赖变化都需检查 |
| 粗化合并 | 锁过多时会自动提升粒度,可能增加误报(false positive) |
| **误报** | 检测出"可能"的环就终止事务,存在假阳性 → 需要重试 |
| 推广风险 | 长事务 + 大量读 → 谓词锁膨胀,可能触发日志警告 |

```bash
# 日志中可能出现
WARNING: reducing predicate locking due to low memory
HINT: Consider increasing max_pred_locks_per_transaction.
```

## ③ 源码路径

| 内容 | 位置 |
|------|------|
| SSI 主实现 | `src/backend/storage/lmgr/predicate.c` |
| SIREAD 锁 | 同文件(`PredicateLockTuple`, `PredicateLockPage`, ...) |
| 冲突检测 | 同文件(`OnConflict_CheckForSerializationFailure`) |
| 依赖图 | 同文件(`SERIALIZABLEXACT`, `SERIALIZABLEXID`) |
| 隔离级别定义 | `src/include/access/xact.h` |
| 参数 | `src/backend/utils/misc/guc_tables.c` |

## ④ 实验验证

### 实验 1:复现写偏斜(REPEATABLE READ 下不报错但结果错)

```sql
DROP TABLE IF EXISTS on_call;
CREATE TABLE on_call (doctor text PRIMARY KEY, shift text, on_duty boolean);
INSERT INTO on_call VALUES ('Dr.X','A',true), ('Dr.Y','A',true), ('Dr.Z','B',true);

-- 会话 A
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM on_call WHERE shift='A' AND on_duty;   -- 2

-- 会话 B
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM on_call WHERE shift='A' AND on_duty;   -- 2

-- 会话 A
UPDATE on_call SET on_duty=false WHERE doctor='Dr.X';
COMMIT;

-- 会话 B
UPDATE on_call SET on_duty=false WHERE doctor='Dr.Y';
COMMIT;      -- ✅ 成功!但结果违反了"至少 1 人在岗"

SELECT * FROM on_call WHERE shift='A';
-- Dr.X=false, Dr.Y=false  ← 0 人在岗,约束被破坏
```

### 实验 2:同样的场景在 SERIALIZABLE 下被阻止

```sql
-- 重置数据
UPDATE on_call SET on_duty=true;

-- 会话 A
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call WHERE shift='A' AND on_duty;   -- 2

-- 会话 B
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call WHERE shift='A' AND on_duty;   -- 2

-- 会话 A
UPDATE on_call SET on_duty=false WHERE doctor='Dr.X';
COMMIT;      -- 成功

-- 会话 B
UPDATE on_call SET on_duty=false WHERE doctor='Dr.Y';
COMMIT;
-- ERROR: could not serialize access due to read/write dependencies among transactions
-- SQLSTATE: 40001   ← 必须重试
```

**对比这两个实验,就理解了 SSI 的价值。**

### 实验 3:观察谓词锁

```sql
-- 会话 A(保持事务打开)
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call WHERE shift='A';

-- 会话 B
SELECT locktype, mode, granted, relation::regclass, page, tuple
FROM pg_locks
WHERE locktype = 'predicate'
   OR mode LIKE '%SIREAD%';
-- 可能看到 predicate 类型的 SIREAD 锁
-- 注意:视图对 predicate lock 的展示有限,部分信息在内部结构中
```

### 实验 4:验证误报(假阳性)

```sql
-- 两个事务操作"不冲突"的数据,但可能仍被判定冲突
-- 会话 A
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call WHERE shift='A';

-- 会话 B
BEGIN ISOLATION LEVEL SERIALIZABLE;
UPDATE on_call SET on_duty=true WHERE shift='B';   -- 不同 shift
COMMIT;
-- 可能仍然成功,也可能被判定冲突(取决于谓词锁粒度粗化程度)
```

### 实验 5:`SERIALIZABLE READ ONLY DEFERRABLE` 不失败

```sql
-- 长报表的安全模式
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SELECT shift, count(*) FROM on_call GROUP BY shift;
-- 这个事务不会因串行化冲突失败(开始时会等待安全快照)
COMMIT;
```

### 实验 6:验证只读事务在普通 SERIALIZABLE 下可能失败

```sql
-- 会话 A(长只读)
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY;      -- 注意:没有 DEFERRABLE
SELECT * FROM on_call;

-- 会话 B 做写操作并提交,若干次后
-- 会话 A 的后续查询或 COMMIT 可能报 40001
```

## ⑤ 生产启示

| 结论 | 行动 |
|------|------|
| SI 不能防写偏斜 | **涉及"读后写"的一致性约束必须用 `SERIALIZABLE`** |
| SSI 不阻塞,只检测 | 并发性优于表锁,代价是共享内存与检测开销 |
| 失败必须重试**整个事务** | 只重试单条语句是错误做法 |
| 重试需指数退避 | 避免"重试风暴" |
| 事务要短 | SSI 开销与事务长度正相关;长事务会积累大量谓词锁 |
| 只读长报表用 `DEFERRABLE` | `SERIALIZABLE READ ONLY DEFERRABLE` 可避免失败 |
| 内存警告要重视 | 日志出现 `reducing predicate locking` 需调大 `max_pred_locks_per_transaction` |
| 误报是设计特性 | 有假阳性;不能假设"没冲突就一定成功" |

```sql
-- 谓词锁相关参数
SHOW max_pred_locks_per_transaction;
SHOW max_pred_locks_per_relation;
SHOW max_pred_locks_per_page;

-- 监控序列化失败(应用侧也需统计)
SELECT datname, xact_rollback FROM pg_stat_database;

-- 当前运行事务的隔离级别
SELECT pid, state, xact_start, left(query,50)
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

## 相关页面

- [[postgresql/03-进阶阶段-深度剖析/并发控制与锁内核/原理剖析-锁管理器与死锁]]
- [[postgresql/03-进阶阶段-深度剖析/事务与MVCC内核/原理剖析-MVCC与快照隔离]]
- [[postgresql/01-入门阶段-基础教程/第06章-事务与权限入门/6.1-事务基础]]
