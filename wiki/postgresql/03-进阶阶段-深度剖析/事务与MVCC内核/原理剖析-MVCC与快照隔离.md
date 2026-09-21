---
tags: [concept, PostgreSQL, 进阶, MVCC, 快照, 隔离, 事务]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# 原理剖析 · MVCC 与快照隔离

## ① 现象

- 读写互不阻塞:一个事务在改数据,另一个事务仍能读旧值
- `REPEATABLE READ` 下同一事务重复查询结果不变
- `UPDATE` 后表膨胀(每个版本都留在磁盘上)
- 长事务会让表越来越大、VACUUM 失效
- 备库上长查询会被取消(`canceling statement due to conflict with recovery`)

## ② 机制

### MVCC 的核心思想

**不覆盖旧数据,而是为每次修改产生新版本**,并用"事务可见性规则"决定某个快照能看到哪些版本。

PostgreSQL 的实现属于**追加式 MVCC(append-only MVCC)**,与 Oracle/MySQL 的 undo-log 式不同:

| 维度 | PostgreSQL | undo 式(Oracle/MySQL) |
|------|-----------|----------------------|
| 新版本位置 | 写在新物理位置(新元组) | 通常在原页,旧版本进 undo |
| 旧版本位置 | 留在原页(标记失效) | 移到 undo 段 |
| 读旧版本 | 直接读原页 | 需回滚构造 |
| 副作用 | **表膨胀 + 需 VACUUM** | 不需 VACUUM,但 undo 会涨 |
| 更新行时 | 需更新所有索引项(除 HOT) | 索引稳定性更好 |

### 快照(Snapshot)

**快照定义了"可见性边界"**,核心是三个字段:

```c
/* src/include/utils/snapshot.h */
typedef struct SnapshotData
{
    TransactionId xmin;       /* 小于它的 xid 都已结束 → 可见性只看是否已提交 */
    TransactionId xmax;       /* 大于等于它的 xid 都未结束 → 一定不可见 */
    TransactionId *xip;       /* 正在进行的 xid 列表(在 xmin~xmax 之间) */
    uint32        xcnt;       /* xip 列表长度 */
    CommandId     curcid;     /* 命令 ID,处理同事务内的多命令可见性 */
    ...
} SnapshotData;
```

**可见性判定规则(简化)** — 对一个元组的 `xmin`:

```
若 xmin == 本事务的 xid
    → 由 curcid 决定(同一事务中"当前命令之后"的插入不可见)
否则若 xmin < snapshot.xmin
    → 若 xmin 已提交 → 可见
    → 若 xmin 已回滚/中止 → 不可见
否则若 xmin >= snapshot.xmax
    → 不可见(该事务在快照之后才启动)
否则若 xmin 在 xip 列表中
    → 不可见(快照时该事务仍在进行)
否则
    → 若已提交 → 可见,否则不可见
```

对 `xmax` 做类似判定:若 `xmax` 非 0 且"对快照可见"(即删除已生效),则该版本不可见。

### 快照的类型

| 类型 | 获取时机 | 用途 |
|------|----------|------|
| **语句快照**(statement snapshot) | 每条语句开始时 | `READ COMMITTED` 用这个 |
| **事务快照**(transaction snapshot) | 事务的第一条语句 | `REPEATABLE READ` / `SERIALIZABLE` 用这个 |
| self snapshot | 用于判断自己事务内的变更 | 配合 `curcid` |
| historic snapshot | 逻辑解码用 | 复制 |

> **`READ COMMITTED` 与 `REPEATABLE READ` 的全部差异,就在于"何时取快照"**:
> - `READ COMMITTED` 每条语句取一次 → 同一事务内两次查询可能不同
> - `REPEATABLE READ` 事务开始时取一次,全程复用 → 稳定

### 隔离级别的实现映射

| 声明级别 | PostgreSQL 实现 |
|----------|-----------------|
| `READ UNCOMMITTED` | 降级为 `READ COMMITTED`(不支持脏读) |
| `READ COMMITTED` | 语句级快照 |
| `REPEATABLE READ` | 事务级快照(**快照隔离 SI**) |
| `SERIALIZABLE` | SI + SSI(可串行化快照隔离,加依赖检测) |

> **PostgreSQL 的 `REPEATABLE READ` 实际上提供了"快照隔离(SI)"**,这比 SQL 标准要求更强:它不仅避免不可重复读,还**避免了幻读**(标准只要求 `SERIALIZABLE` 才避免幻读)。但 SI 仍允许**写偏斜(write skew)**,需要 `SERIALIZABLE` 才能排除。

### 快照隔离的异常

| 异常 | SI 是否阻止 | 举例 |
|------|-------------|------|
| 脏读 | ✅ | — |
| 不可重复读 | ✅ | — |
| 幻读 | ✅ | — |
| 丢失更新 | ✅(会报串行化错误) | 两个事务读同一值再各自 +1 |
| **写偏斜** | ❌ | 两个事务各查"是否还有人在岗",各自认为自己可离岗 |
| 只读事务异常 | ❌ | 只读事务可能看到不一致的组合 |

写偏斜的经典形式:

```
约束:至少 1 名医生在岗
事务 A: SELECT count(*) WHERE on_duty   → 2  (认为可以离岗)
事务 B: SELECT count(*) WHERE on_duty   → 2  (认为可以离岗)
事务 A: UPDATE 自己 on_duty = false      → 提交
事务 B: UPDATE 自己 on_duty = false      → 提交
结果:0 人在岗 —— 约束被违反
```

**SI 无法捕捉**:A 与 B 读的是**不同行**、写的是**不同行**,没有冲突。只有 SSI(检测读写依赖环)才能发现。见 [[postgresql/03-进阶阶段-深度剖析/并发控制与锁内核/原理剖析-SSI可串行化隔离]]。

### 谁负责清理旧版本

| 机制 | 作用 |
|------|------|
| `VACUUM` | 回收"对所有快照都不可见"的死元组 |
| `autovacuum` | 自动触发上述 |
| 冻结(freeze) | 把老元组的 `xmin` 标记为 `FrozenTransactionId`,使得 xid 回绕后仍可见 |
| `pg_xact`(CLOG) | 记录每个事务的最终状态(提交/中止),是可见性判定的依据 |

**长事务为什么灾难性**:它持有的快照让"旧版本"仍可能被需要,VACUUM 不敢回收。于是死元组不断累积,直到该事务结束。

## ③ 源码路径

| 内容 | 位置 |
|------|------|
| 快照结构定义 | `src/include/utils/snapshot.h` |
| 快照获取 | `src/backend/utils/time/snapmgr.c` |
| 可见性判定 | `src/backend/access/heap/heapam_visibility.c`(`HeapTupleSatisfiesMVCC` 等) |
| 事务状态 | `src/backend/access/transam/clog.c` |
| VACUUM | `src/backend/access/heap/vacuumltd.c` |
| 事务 ID 分配 | `src/backend/access/transam/xid.c`, `varsup.c` |
| 子事务 | `src/backend/access/transam/subtrans.c` |

## ④ 实验验证

### 实验 1:手工判定可见性

```sql
DROP TABLE IF EXISTS mvcc_demo;
CREATE TABLE mvcc_demo (id int, v text);
INSERT INTO mvcc_demo VALUES (1, 'a');

-- 当前事务 ID
SELECT txid_current();

SELECT ctid, xmin, xmax, v FROM mvcc_demo;
-- xmin = 插入事务的 xid,xmax = 0

-- 查看自己看到的快照
SELECT pg_current_snapshot();
-- 形如 1000:1010:1000,1002  → xmin:xmax:xip
```

**`pg_current_snapshot()` 的格式**:`xmin:xmax:xip1,xip2`。可直接看到快照的边界与进行中的事务列表。

```sql
-- 用系统函数判断某个元组对某快照是否可见
SELECT pg_visible_in_snapshot(xmin, pg_current_snapshot()) FROM mvcc_demo;
-- false:因为它是本事务自己插入的,走 curcid 逻辑
```

### 实验 2:复现"读写不阻塞"

需要一个真正并发的场景。用两个 psql 会话:

```sql
-- 会话 A
BEGIN;
UPDATE mvcc_demo SET v = 'A-修改中' WHERE id = 1;
-- 不提交,持锁

-- 会话 B(另一个连接)
SELECT * FROM mvcc_demo;              -- ✅ 立刻返回旧值 'a'(未被阻塞)
UPDATE mvcc_demo SET v = 'B' WHERE id = 1;   -- ⏳ 阻塞(等行锁)
```

**结论**:读不阻塞(靠 MVCC 快照),写阻塞(靠行锁)。

### 实验 3:`READ COMMITTED` vs `REPEATABLE READ`

```sql
-- 会话 A
BEGIN;                                        -- 默认 READ COMMITTED
SELECT v FROM mvcc_demo WHERE id = 1;          -- 记录结果

-- 会话 B
UPDATE mvcc_demo SET v = 'B-changed' WHERE id = 1;

-- 会话 A
SELECT v FROM mvcc_demo WHERE id = 1;          -- ⚠️ 变新了(语句级新快照)
COMMIT;
```

```sql
-- 换成 REPEATABLE READ
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT v FROM mvcc_demo WHERE id = 1;
-- (会话 B 修改并提交)
SELECT v FROM mvcc_demo WHERE id = 1;          -- ✅ 仍是旧值(事务级快照)
COMMIT;
```

### 实验 4:观察死元组与长事务阻塞 VACUUM

```sql
-- 会话 A:开一个长事务并保持
BEGIN;
SELECT count(*) FROM mvcc_demo;

-- 会话 B:制造死元组
UPDATE mvcc_demo SET v = v || '!' FROM generate_series(1,1000);

-- 会话 B:尝试 VACUUM
VACUUM VERBOSE mvcc_demo;
-- 观察输出中的 "there are no unremovable dead tuples" 类信息与实际清除数量

-- 会话 A:提交
COMMIT;
-- 会话 B:再 VACUUM 一次,可清除数量明显增加
VACUUM VERBOSE mvcc_demo;
```

### 实验 5:观察 xid 回绕防护

```sql
-- 查看数据库的年龄(距回绕的剩余事务数)
SELECT datname, age(datfrozenxid) AS xid_age, datfrozenxid
FROM pg_database ORDER BY age(datfrozenxid) DESC;

-- 严重警告阈值
SHOW autovacuum_freeze_max_age;      -- 默认 200000000
SHOW vacuum_freeze_min_age;
SHOW vacuum_freeze_table_age;
```

### 实验 6:观察 hint bits 的影响

```sql
-- 未提示的行,第一次读取需查 pg_xact,并写回 hint bits
DROP TABLE IF EXISTS hint_demo;
CREATE TABLE hint_demo AS SELECT g AS id FROM generate_series(1, 100000) g;
-- 立即查,注意可能需要读 pg_xact
SELECT t_infomask FROM heap_page_items(get_raw_page('hint_demo', 0)) LIMIT 5;
-- 观察 XMIN_COMMITTED 位
```

## ⑤ 生产启示

| 结论 | 行动 |
|------|------|
| 读不阻塞写、写不阻塞读 | 读多写少场景并发能力好;但**写写同一行仍会阻塞** |
| 每个版本都留盘 | **避免无意义的大批量 `UPDATE`**(如全表刷新字段),它会造成巨量死元组 |
| 长事务是最大隐患 | 必须配 `idle_in_transaction_session_timeout`;事务要短 |
| `REPEATABLE READ` 已避免幻读 | 需要稳定读视图时用它,不必直接上 `SERIALIZABLE` |
| `SERIALIZABLE` 需重试 | 遇 SQLSTATE `40001` 必须重试整个事务 |
| 冻结与回绕是真风险 | 监控 `age(datfrozenxid)`;`autovacuum` 不能被长期阻塞 |
| 备库查询可能被取消 | 用 `hot_standby_feedback` 或 `max_standby_streaming_delay` 权衡 |

```sql
-- 长事务 / 老快照监控 ★必配
SELECT pid, state, xact_start, now() - xact_start AS xact_age,
       backend_xmin, left(query, 50)
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;

-- 回绕风险监控
SELECT datname, age(datfrozenxid) AS age FROM pg_database ORDER BY 2 DESC;
SELECT relname, age(relfrozenxid) AS age FROM pg_class WHERE relkind='r' ORDER BY 2 DESC LIMIT 10;
```

## 相关页面

- [[postgresql/03-进阶阶段-深度剖析/事务与MVCC内核/原理剖析-可见性与事务状态]]
- [[postgresql/03-进阶阶段-深度剖析/并发控制与锁内核/原理剖析-SSI可串行化隔离]]
- [[postgresql/01-入门阶段-基础教程/第06章-事务与权限入门/6.1-事务基础]]
