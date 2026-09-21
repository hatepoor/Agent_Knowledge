---
tags: [concept, PostgreSQL, 进阶, 索引内核, B-tree, 页结构, 去重, skip scan]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# 原理剖析 · B-tree 结构

## ① 现象

- 索引比表还大
- `REINDEX` 后查询变快
- 索引页出现大量空白(`avg_leaf_density` 低)
- 相同数据,多列索引的顺序导致性能差异巨大
- PG18 之后某些"前导列无等值条件"的查询突然能用索引了

## ② 机制

### B-tree 的页类型

```
                   ┌──────────────┐
                   │  metapage    │  第 0 页:记录 root 页位置、版本
                   │  (block 0)   │
                   └──────┬───────┘
                          │
                   ┌──────▼───────┐
                   │   root page  │  根(内部节点)
                   └──┬────────┬──┘
                      │        │
             ┌────────▼──┐  ┌──▼────────┐
             │ internal  │  │ internal  │  内部节点(可多层)
             └──┬─────┬──┘  └──┬─────┬──┘
                │     │        │     │
           ┌────▼┐ ┌──▼──┐ ┌───▼─┐ ┌─▼───┐
           │leaf │ │leaf │ │leaf │ │leaf │  叶子页:存放索引项 + ctid
           └─────┘ └─────┘ └─────┘ └─────┘
              ↕ (由右链 right-link 串联)
```

### 索引页的 special space

B-tree 页的 `pd_special` 区域存放 `BTPageOpaqueData`:

```c
/* src/include/access/nbtree.h */
typedef struct BTPageOpaqueData
{
    BlockNumber btpo_prev;      /* 左兄弟页 */
    BlockNumber btpo_next;      /* 右兄弟页(right-link) */
    uint32      btpo_level;     /* 0 = 叶子层 */
    uint16      btpo_flags;     /* 页类型:LEAF / ROOT / INTERNAL / META / DELETED / HALF_DEAD */
    BTCycleId   btpo_cycleid;
} BTPageOpaqueData;
```

| 字段 | 意义 |
|------|------|
| `btpo_prev` / `btpo_next` | 同层页的双向链(支持范围扫描与并发分裂) |
| `btpo_level` | 层号,0 为叶子 |
| `btpo_flags` | 页角色 |

### 叶子页的索引项(IndexTuple)

```
IndexTupleHeader (t_tid + t_info)
   │
   ├─ t_tid:ctid  → 指向堆表中的行指针(块号, 槽号)
   └─ 索引键值(一个或多个列)
```

两个关键点:

1. **索引项存的是 `ctid`,不是数据本身**。因此索引扫描需"回表"取其他列(除非是覆盖索引 / 仅索引扫描)。
2. **`ctid` 指向行指针而非元组**。所以元组在页内移动时,只需改行指针,索引无需改 —— 这是 HOT 更新的基础。

### 三个核心结构特性

**① 去重(deduplication,PG13+)**

叶子页中"相同键值 + 相同 posting list"的索引项会被合并为一个**posting tuple**,其中记录多个 `ctid`。

```
去重前:  [key=5, tid=(0,1)] [key=5, tid=(0,2)] [key=5, tid=(0,3)] ...
去重后:  [key=5, posting list: (0,1),(0,2),(0,3), ...]
```

| 收益 | 代价 |
|------|------|
| 显著缩小索引(低基数列尤其明显) | 取多个 ctid 有少量解析开销 |
| 减少页分裂 | |
| 提高缓存命中 | |

**② 高键(high key)**

每个非根页都存放一个"高键"= 该页允许的最大键值 + 1。作用:

- 快速判断某键是否属于该页范围
- 支持并发分裂时判断"是否需向右走"

**③ 右链 + Lehman-Yao 算法**

并发分裂时,不阻塞读:

```
1. 分裂页 P 为 P 与 Q
2. 先把 Q 链接到 P 的右侧(P.next = Q)
3. 再把高键写入 P
4. 最后在父节点插入指向 Q 的条目
```

读操作若发现"目标键 > P 的高键",就沿 `btpo_next` 向右查找 —— 即使在分裂中间状态也能得到正确结果。**这是 B-tree 能支持高并发写入的关键。**

### 多列索引的比较语义

`(a, b, c)` 的索引按**字典序**排序:

```
比较顺序:先比 a,再比 b,再比 c
```

这就是"最左前缀"规则的来源:索引的物理有序性只保证"按 a 排序,同 a 内按 b",所以只有 `a`(以及 `a` 后的连续前缀)能被用于定位。

### Skip Scan(PG18 新增)

**问题**:索引 `(a, b)`,`WHERE b = 5`(无 `a` 条件)以前无法用索引。

**PG18 的做法**:在索引内部枚举 `a` 的不同取值,对每个取值做一次"`a = x AND b = 5`"的定位。

```
等效于对 a 的每个 distinct 值:
    Index Cond: (a = 'A1') AND (b = 5)
    Index Cond: (a = 'A2') AND (b = 5)
    ...
```

**收益条件**

| 条件 | 说明 |
|------|------|
| 前导列基数**低** | distinct 值少,枚举代价小 |
| 后续列选择性好 | 每次定位能快速收敛 |
| 前导列值分布不过于倾斜 | 否则个别值成本高 |

> **实践含义**:以前"必须把高频过滤列放最左"的规则在 PG18 下可以放宽。但**不要因此重建所有索引** —— Skip Scan 只在低基数前导列时显著优于全扫。仍应优先按最左前缀设计。

### 页分裂与填充因子

```
页满 → 分裂为两页 → 父节点增加条目 → 可能级联分裂到根
```

`fillfactor` 控制分裂后的填充目标:

```sql
-- 默认 90(索引)
CREATE INDEX idx ON t (col) WITH (fillfactor = 70);
-- 低 fillfactor → 留更多空间 → 长时间不需再分裂 → 适合随机写入
```

**随机插入 vs 顺序插入**

| 插入模式 | 后果 |
|----------|------|
| 顺序(自增主键、`uuidv7()`) | 总在最右页追加,分裂少 |
| 随机(`uuidv4()`、随机字符串) | 均匀命中所有页,频繁分裂,密度低 |

**这正是 PG18 引入 `uuidv7()` 的动机**:让 UUID 主键获得时间递增性,恢复索引局部性。

## ③ 源码路径

| 内容 | 位置 |
|------|------|
| B-tree 页结构定义 | `src/include/access/nbtree.h`(`BTPageOpaqueData`, `BTMaxItemSize`) |
| 插入与分裂 | `src/backend/access/nbtree/nbtinsert.c` |
| 查找 | `src/backend/access/nbtree/nbtsearch.c` |
| 分裂的 Lehman-Yao 实现 | `src/backend/access/nbtree/nbtpage.c`(`_bt_split`) |
| 去重 | `src/backend/access/nbtree/nbtdedup.c` |
| VACUUM 修剪 | `src/backend/access/nbtree/nbtree.c`, `nbtpage.c`(`_bt_pagedel`) |
| Skip Scan | `src/backend/access/nbtree/nbtsearch.c`(`_bt_skip_scan` 相关) |
| IndexTuple 定义 | `src/include/access/itup.h` |

## ④ 实验验证

### 实验 1:看索引的层次与密度

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS idx_demo;
CREATE TABLE idx_demo AS SELECT g AS id, (g % 1000) AS k FROM generate_series(1, 2000000) g;
CREATE INDEX idx_demo_k ON idx_demo (k);
VACUUM ANALYZE idx_demo;

SELECT * FROM pgstatindex('idx_demo_k');
-- version | tree_level | index_size | root_block_no | internal_pages | leaf_pages |
-- empty_pages | deleted_pages | avg_leaf_density | leaf_fragmentation
```

**关注字段**

| 字段 | 含义 | 期望 |
|------|------|------|
| `tree_level` | 树高(1 表示只有叶层) | 一般 2–4 |
| `avg_leaf_density` | 叶子页平均填充率 | 接近 90%(fillfactor) |
| `leaf_fragmentation` | 叶页的物理顺序与逻辑顺序偏离度 | 低则顺序扫描友好 |

### 实验 2:读索引页内容

```sql
-- 第 0 页是 metapage
SELECT * FROM bt_metap('idx_demo_k');
-- magic | version | root | level | fastroot | fastlevel | ...

-- 读叶子/内部页
SELECT * FROM bt_page_items('idx_demo_k', 1) LIMIT 10;

-- 页统计
SELECT * FROM bt_page_stats('idx_demo_k', 1);
-- 关注 type(l=leaf, i=internal, r=root, m=metapage)、
--      live_items、dead_items、avg_item_size、free_size
```

### 实验 3:观察去重效果

```sql
-- 低基数列(去重收益大)
DROP TABLE IF EXISTS dedup_demo;
CREATE TABLE dedup_demo AS SELECT g AS id, (g % 10) AS low FROM generate_series(1, 1000000) g;
CREATE INDEX dedup_idx ON dedup_demo (low);
ANALYZE dedup_demo;
SELECT pg_size_pretty(pg_relation_size('dedup_idx')) AS deduped_size;

-- 关闭去重(需重建索引)
CREATE INDEX dedup_idx_nodup ON dedup_demo (low) WITH (deduplicate_items = off);
SELECT pg_size_pretty(pg_relation_size('dedup_idx_nodup')) AS nodedup_size;
-- 差异明显(可能数倍)
```

### 实验 4:观察随机 vs 顺序插入的索引密度

```sql
-- 顺序插入
DROP TABLE IF EXISTS seq_ins;
CREATE TABLE seq_ins (id bigint, v text);
CREATE INDEX seq_ins_id ON seq_ins (id);
INSERT INTO seq_ins SELECT g, 'x' FROM generate_series(1, 500000) g;
VACUUM ANALYZE seq_ins;
SELECT 'sequential' AS mode, avg_leaf_density, leaf_fragmentation FROM pgstatindex('seq_ins_id');

-- 随机插入
DROP TABLE IF EXISTS rnd_ins;
CREATE TABLE rnd_ins (id bigint, v text);
CREATE INDEX rnd_ins_id ON rnd_ins (id);
INSERT INTO rnd_ins SELECT (random()*500000)::bigint, 'x' FROM generate_series(1, 500000);
VACUUM ANALYZE rnd_ins;
SELECT 'random' AS mode, avg_leaf_density, leaf_fragmentation FROM pgstatindex('rnd_ins_id');
```

### 实验 5:验证最左前缀

```sql
DROP TABLE IF EXISTS prefix_demo;
CREATE TABLE prefix_demo AS SELECT g AS a, (g%100) AS b, (g%10) AS c FROM generate_series(1,1000000) g;
CREATE INDEX prefix_idx ON prefix_demo (a, b, c);
ANALYZE prefix_demo;

-- 能用
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM prefix_demo WHERE a = 500;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM prefix_demo WHERE a = 500 AND b = 5;

-- 不能用(跳过 a)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM prefix_demo WHERE b = 5;

-- PG18:最后一个可能被 Skip Scan 使用!(而 b 基数 100,可能仍选 Seq Scan)
-- 判断标准:看计划里是 Index Scan with Skip Scan 还是 Seq Scan
```

### 实验 6:验证 fillfactor

```sql
CREATE INDEX ff_idx ON prefix_demo (c) WITH (fillfactor = 50);
VACUUM ANALYZE prefix_demo;
SELECT avg_leaf_density, leaf_fragmentation FROM pgstatindex('ff_idx');
-- avg_leaf_density 应接近 50
```

### 实验 7:索引膨胀与 REINDEX

```sql
-- 制造索引膨胀
CREATE INDEX bloat_idx ON prefix_demo (b);
UPDATE prefix_demo SET b = (b + 1) % 100 WHERE a % 2 = 0;

SELECT pg_size_pretty(pg_relation_size('bloat_idx')) AS before_reindex,
       avg_leaf_density FROM pgstatindex('bloat_idx');

REINDEX INDEX bloat_idx;

SELECT pg_size_pretty(pg_relation_size('bloat_idx')) AS after_reindex,
       avg_leaf_density FROM pgstatindex('bloat_idx');
```

> **关键洞察**:`UPDATE` 涉及索引列时,旧索引项**不会立即移除**,只能标记为 dead,等 VACUUM 清理。因此索引膨胀通常比表膨胀更"顽固"。

## ⑤ 生产启示

| 结论 | 行动 |
|------|------|
| 索引项指向行指针 | HOT 更新可行;但改索引列时必须新增索引项 |
| 低基数列去重收益大 | PG13+ 默认开启,不要关闭 |
| 随机键导致索引稀疏 | **主键优先顺序性**:自增 ID 或 PG18 的 `uuidv7()` |
| 索引膨胀需主动处理 | 监控 `avg_leaf_density`;必要时 `REINDEX CONCURRENTLY` |
| 多列顺序决定可用范围 | 仍按"等值列在前、范围列在后"设计;PG18 Skip Scan 是补充而非替代 |
| 索引比为表还大是可能的 | 用 `pg_total_relation_size` 与 `pgstatindex` 量化 |
| HOT 比例是健康指标 | `n_tup_hot_upd / n_tup_upd` 低说明更新总触发索引维护 |

```sql
-- 索引健康检查
SELECT c.relname AS table, i.relname AS index,
       pg_size_pretty(pg_relation_size(i.oid)) AS idx_size,
       s.idx_scan,
       (pgstatindex(i.oid)).avg_leaf_density AS density,
       (pgstatindex(i.oid)).leaf_fragmentation AS frag
FROM pg_index x
JOIN pg_class c ON c.oid = x.indrelid
JOIN pg_class i ON i.oid = x.indexrelid
LEFT JOIN pg_stat_user_indexes s ON s.indexrelid = i.oid
WHERE c.relkind = 'r' AND i.relkind = 'i'
ORDER BY pg_relation_size(i.oid) DESC LIMIT 15;
```

```sql
-- HOT 更新健康度
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
FROM pg_stat_user_tables
WHERE n_tup_upd > 0 ORDER BY hot_pct LIMIT 10;
-- hot_pct 低 → 索引设计可能过宽(很多列被索引,导致 HOT 无法生效)
```

## 相关页面

- [[postgresql/03-进阶阶段-深度剖析/索引内核/原理剖析-GIN-GiST-BRIN对比]]
- [[postgresql/03-进阶阶段-深度剖析/索引内核/原理剖析-HOT更新与索引维护]]
- [[postgresql/02-中段阶段-专题精讲/索引实战应用/索引类型选型与设计]]
