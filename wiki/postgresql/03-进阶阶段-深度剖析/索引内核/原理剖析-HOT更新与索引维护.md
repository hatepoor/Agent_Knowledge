---
tags: [concept, PostgreSQL, 进阶, HOT更新, 索引维护, fillfactor, 索引膨胀]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# 原理剖析 · HOT 更新与索引维护

## ① 现象

- 只改了一个非索引列,却导致所有索引被维护
- `n_tup_hot_upd` 比例很低(表更新几乎都触发索引写)
- 索引膨胀比表膨胀更严重、更难清理
- 加了 `fillfactor` 却没看到效果
- `REINDEX` 能让查询显著变快

## ② 机制

### 问题:更新为什么要动索引

回忆堆表的第一个事实:**索引项存的是 `ctid`(指向行指针)**。

```
B-tree 叶子:  [key='A'] → ctid(0,5)
堆表页 0:     lp5 → 元组(xmin=100, v='a')
```

现在 `UPDATE t SET v='b'`:

```
非 HOT 方案(朴素做法):
1. 在页 3 的新位置写入新元组
2. 新元组的 ctid = (3, 8)
3. 索引里 [key='A'] 的 ctid 必须从 (0,5) 改成 (3,8)  ← 每个索引都要改!
```

**问题**:即使被更新的列完全没被索引,也必须在**每个索引**里新增一个条目。

- 代价:写放大(1 次行更新 → N 次索引插入)
- 后果:索引膨胀、WAL 放大、页分裂

### HOT:Heap-Only Tuple

**核心洞察**:如果没有索引列被修改,那么索引仍需指向"逻辑上的同一行"。既然索引指向的是**行指针(lp)**,那么只要**让行指针指向新位置**,索引就完全不需要改动。

```
HOT 更新的机制:

更新前:
  堆表页 0:  lp5 → 元组A(xmin=100, v='a')        ← 索引指向 lp5
  索引:      [key='A'] → ctid(0,5)

UPDATE t SET v='b'(v 未被索引):

  步骤 1:在同一页(或新页)写入新元组 B,ctid=(0,9)
  步骤 2:把 lp5 改为 LP_REDIRECT 状态,指向 lp9  ← 只改行指针!
  步骤 3:元组 A 的 t_ctid 指向 (0,9)              ← 形成版本链
  步骤 4:元组 B 的 t_ctid 指向自己 (0,9)

更新后:
  堆表页 0:  lp5 → REDIRECT → lp9
             lp9 → 元组B(xmin=101, v='b')
  索引:      [key='A'] → ctid(0,5)   ← 完全没变!
```

**关键条件(必须全部满足)**

| 条件 | 说明 |
|------|------|
| 没有索引列被修改 | 任何一个索引列变了 → 必须走非 HOT |
| **新版本能放进同一页** | 页内需有足够空闲空间;否则必须跨页 → 非 HOT |
| 没触发页分裂 | 同上 |

第三个条件解释了 **`fillfactor` 的作用**:

```sql
-- 默认 fillfactor = 100(表),即页填满为止
-- HOT 需要页内空间,所以写密集表应留空
ALTER TABLE orders SET (fillfactor = 80);
CREATE INDEX ... ;   -- 索引默认 fillfactor = 90
```

### 版本链与清理

HOT 更新产生的链:

```
lp5(REDIRECT) → lp9(DEAD) → lp12(DEAD) → lp15(LIVE)
```

`VACUUM` 时:

1. 把死链上的行指针标记为 `LP_UNUSED`
2. 若链头(索引指向的那个)对应的元组已死,则**保留一个重定向指针**指向活元组
3. 若整个链都死,行指针可完全复用

再加上 PG14+ 的**索引页删除**能力,VACUUM 还能把索引里指向死元组的条目清理掉(此前只能标记 dead,不能物理删除)。

### 页修剪(page pruning)

HOT 更新时若页内空闲空间不足,会先触发 `heap_page_prune`:尝试清理页内的死元组腾出空间。

```c
pd_prune_xid    /* 页头字段:最早可修剪的 xid */
```

若修剪后空间仍不够 → 必须走非 HOT(或触发页分裂)。

### 索引膨胀的成因

| 成因 | 机制 |
|------|------|
| `UPDATE` 索引列 | 每次新增索引项,旧项变 dead |
| 非 HOT 更新 | 同上 |
| `DELETE` | 索引项变 dead,等 VACUUM |
| 随机键插入 | 页分裂,每页填充率低 |
| VACUUM 未及时 | dead 项累积 |

**关键差异:索引膨胀比表膨胀更难治**

| 对象 | VACUUM 能做什么 |
|------|-----------------|
| 表 | 把死元组空间标记为可复用(页内) |
| 索引 | PG14+ 能物理删除 dead 项;此前只能标记,页不回收 |
| 两者 | `VACUUM FULL` / `REINDEX` 才能真正缩小文件 |

> 所以在 PG14 之前,"索引膨胀"往往只能靠 `REINDEX` 解决。这是 PG14 索引页删除(由 `VACUUM` 触发)的重要改进。

### 索引维护的相关参数

| 参数 | 默认 | 作用 |
|------|------|------|
| 表 `fillfactor` | 100 | 表页预留空间(为 HOT 与更新留位) |
| 索引 `fillfactor` | 90 | 索引页预留空间,减少分裂 |
| `deduplicate_items` | on | B-tree 去重 |
| `autovacuum_...` | — | 触发时机 |
| `maintenance_work_mem` | 64MB | 建索引/`VACUUM` 内存 |
| `max_parallel_maintenance_workers` | 2 | 并行建索引 |

```sql
-- 并行建索引(加速)
SET maintenance_work_mem = '1GB';
SET max_parallel_maintenance_workers = 4;
CREATE INDEX idx ON big_table (col);
```

## ③ 源码路径

| 内容 | 位置 |
|------|------|
| HOT 判定与执行 | `src/backend/access/heap/heapam.c`(`heap_update`, `HeapUpdateHotTuple`) |
| 页修剪 | `src/backend/access/heap/pruneheap.c` |
| 行指针重定向常量 | `src/include/access/htup_details.h`(`LP_REDIRECT`) |
| VACUUM 清理索引 | `src/backend/access/heap/vacuumltd.c`, `src/backend/access/nbtree/nbtree.c` |
| 索引页删除(PG14+) | `src/backend/access/nbtree/nbtpage.c`(`_bt_pagedel`) |
| 统计计数 | `src/backend/utils/adt/pgstatfuncs.c`(`n_tup_hot_upd`) |

## ④ 实验验证

### 实验 1:证明"改非索引列"走 HOT

```sql
DROP TABLE IF EXISTS hot_demo;
CREATE TABLE hot_demo (id int PRIMARY KEY, v text);
INSERT INTO hot_demo SELECT g, 'v' || g FROM generate_series(1, 10000) g;

-- 重置统计
SELECT pg_stat_reset();

-- 只改非索引列 v
UPDATE hot_demo SET v = v || '!' WHERE id % 10 = 0;

SELECT n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
FROM pg_stat_user_tables WHERE relname = 'hot_demo';
-- hot_pct 应接近 100%
```

### 实验 2:证明"改索引列"不走 HOT

```sql
SELECT pg_stat_reset();

-- id 是主键(有索引)
UPDATE hot_demo SET id = id + 100000 WHERE id % 10 = 0;

SELECT n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
FROM pg_stat_user_tables WHERE relname = 'hot_demo';
-- hot_pct 应接近 0%
```

### 实验 3:观察行指针 REDIRECT

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS hot_lp;
CREATE TABLE hot_lp (id int PRIMARY KEY, v text);
INSERT INTO hot_lp VALUES (1, 'a');

UPDATE hot_lp SET v = 'b' WHERE id = 1;
UPDATE hot_lp SET v = 'c' WHERE id = 1;

SELECT lp, lp_off, lp_flags, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('hot_lp', 0))
ORDER BY lp;
-- lp_flags: 2 = LP_REDIRECT (链头)
--           3 = LP_DEAD
--           1 = LP_NORMAL (活元组)
```

**预期**:看到第一个行指针是 `REDIRECT`(指向链尾的活元组),中间的 `DEAD`,最后一个是 `NORMAL`。

### 实验 4:验证 fillfactor 提升 HOT 比例

```sql
-- 低 fillfactor:页内留空间,HOT 更容易成功
DROP TABLE IF EXISTS ff_low;
CREATE TABLE ff_low (id int PRIMARY KEY, v text) WITH (fillfactor = 60);
INSERT INTO ff_low SELECT g, 'v'||g FROM generate_series(1, 20000) g;

-- 高 fillfactor(默认 100):页填满,更新可能跨页
DROP TABLE IF EXISTS ff_high;
CREATE TABLE ff_high (id int PRIMARY KEY, v text) WITH (fillfactor = 100);
INSERT INTO ff_high SELECT g, 'v'||g FROM generate_series(1, 20000) g;

-- 对两表做相同的多次更新,比较 HOT 比例与膨胀
SELECT pg_stat_reset();
UPDATE ff_low  SET v = v || repeat('x', 50);
UPDATE ff_high SET v = v || repeat('x', 50);

SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct,
       pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_stat_user_tables WHERE relname IN ('ff_low','ff_high')
GROUP BY 1,2,3,relid;
```

### 实验 5:索引膨胀的量化

```sql
DROP TABLE IF EXISTS ibloat;
CREATE TABLE ibloat AS SELECT g AS id, (g%1000) AS k, 'v'||g AS v FROM generate_series(1, 300000) g;
CREATE INDEX ibloat_k ON ibloat (k);
VACUUM ANALYZE ibloat;

SELECT pg_size_pretty(pg_relation_size('ibloat_k')) AS before_size,
       (pgstatindex('ibloat_k')).avg_leaf_density AS before_density;

-- 反复更新索引列
UPDATE ibloat SET k = (k + 1) % 1000;
UPDATE ibloat SET k = (k + 1) % 1000;
UPDATE ibloat SET k = (k + 1) % 1000;

SELECT pg_size_pretty(pg_relation_size('ibloat_k')) AS after_size,
       (pgstatindex('ibloat_k')).avg_leaf_density AS after_density;
-- 索引变大,密度下降

VACUUM ibloat;
SELECT pg_size_pretty(pg_relation_size('ibloat_k')) AS after_vacuum,
       (pgstatindex('ibloat_k')).avg_leaf_density AS density;
-- 大小基本不变,但 dead_items 减少;PG14+ 密度可能有所恢复

REINDEX INDEX ibloat_k;
SELECT pg_size_pretty(pg_relation_size('ibloat_k')) AS after_reindex,
       (pgstatindex('ibloat_k')).avg_leaf_density AS density;
-- 大小与密度都明显改善
```

### 实验 6:非 HOT 更新的 WAL 放大

```sql
SELECT pg_current_wal_lsn() AS l0 \gset
UPDATE ibloat SET v = v || 'x';        -- 改非索引列
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'l0')::bigint) AS wal_nonidx;

SELECT pg_current_wal_lsn() AS l1 \gset
UPDATE ibloat SET k = (k + 1) % 1000;  -- 改索引列
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'l1')::bigint) AS wal_idx;
-- 后者通常产生更多 WAL
```

## ⑤ 生产启示

| 结论 | 行动 |
|------|------|
| 改索引列 = 全索引维护 | **少建索引**本身就能提升写入性能 |
| 改非索引列可走 HOT | 热更新字段**不要建索引** |
| `fillfactor` 是 HOT 的前提 | 写密集表设 80–90;高更新压力表可更低 |
| 索引不宜过宽 | 索引列越多,HOT 越难生效,写入越慢 |
| 索引膨胀需主动治理 | 监控 `avg_leaf_density` 与 `pgstatindex.deleted_pages` |
| `REINDEX CONCURRENTLY` 可在线修复 | PG12+ 支持,失败会留 invalid 索引需清理 |
| PG14+ 的 VACUUM 能物理删索引页 | 升级后索引膨胀治理难度显著下降 |

```sql
-- HOT 健康度与索引宽度自查
SELECT t.relname,
       t.n_tup_upd, t.n_tup_hot_upd,
       round(100.0*t.n_tup_hot_upd/nullif(t.n_tup_upd,0),1) AS hot_pct,
       (SELECT count(*) FROM pg_index i WHERE i.indrelid = t.relid) AS idx_count,
       pg_size_pretty(pg_total_relation_size(t.relid)) AS total_size
FROM pg_stat_user_tables t
WHERE t.n_tup_upd > 1000
ORDER BY hot_pct LIMIT 15;
-- hot_pct 极低 + idx_count 高 → 索引过宽,考虑精简
```

```sql
-- 找出膨胀严重的索引
SELECT c.relname AS tbl, i.relname AS idx,
       pg_size_pretty(pg_relation_size(i.oid)) AS size,
       (pgstatindex(i.oid)).avg_leaf_density AS density,
       (pgstatindex(i.oid)).deleted_pages AS deleted_pages
FROM pg_class i
JOIN pg_index x ON x.indexrelid = i.oid
JOIN pg_class c ON c.oid = x.indrelid
WHERE i.relkind = 'i'
  AND (pgstatindex(i.oid)).avg_leaf_density < 60
ORDER BY pg_relation_size(i.oid) DESC LIMIT 10;
-- 密度低 + 体积大 → REINDEX 候选
```

```sql
-- 修复流程(在线)
REINDEX INDEX CONCURRENTLY bloat_idx;
ANALYZE tbl;
-- 若曾失败留下 invalid 索引:
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
-- 手工 DROP 后重建
```

## 相关页面

- [[postgresql/03-进阶阶段-深度剖析/索引内核/原理剖析-BTree结构]]
- [[postgresql/03-进阶阶段-深度剖析/存储引擎与页结构/原理剖析-堆表与页结构]]
- [[postgresql/02-中段阶段-专题精讲/索引实战应用/实战-慢查询索引优化]]
