---
tags: [concept, PostgreSQL, 进阶, GIN, GiST, BRIN, SP-GiST, 索引内核]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# 原理剖析 · GIN / GiST / SP-GiST / BRIN

## ① 现象

- JSONB 索引体积巨大,远超数据本身
- GIN 索引写入很慢,批量导入时成为瓶颈
- GiST 索引查询结果里有"假阳性"(需 recheck)
- BRIN 索引小得不可思议,但有时几乎不起作用
- `EXPLAIN` 中出现 `Bitmap Index Scan` + `Recheck Cond`

## ② 机制

### GIN:倒排索引

**结构**:从"元素"指向"包含该元素的行"。

```
GIN 索引(以 text[] 列为例)

  元素(entry)           posting list / posting tree
  ─────────────         ─────────────────────────────
  '数码'         ──►    ctid(0,1) → ctid(0,5) → ctid(1,3) → ...
  '外设'         ──►    ctid(0,1) → ctid(2,7) → ...
  '热销'         ──►    ctid(0,5) → ...
```

**两个组成部分**

| 部分 | 说明 |
|------|------|
| **Entry Tree** | 按元素有序的 B-tree,元素 → posting 的映射 |
| **Posting List / Tree** | 元素对应的 ctid 列表。少量时是 list,多了变成 tree |

**Pending List(快更新机制)**

```c
fastupdate = on(默认)
gin_pending_list_limit = 4MB
```

问题:GIN 的直接插入成本高(要维护 entry tree 与 posting)。解法:**先写入一个无序的 pending list**,查询时同时查主结构与 pending list,当 pending list 超过阈值或 VACUUM 时,再批量合并进主结构。

| 优点 | 缺点 |
|------|------|
| 大幅提升写入速度 | pending list 大时查询变慢(需扫描它) |
| 适合批量导入 | 合并时有额外开销 |

```sql
-- gin_clean_pending_list(索引):手动把 pending list 合并进主索引结构
-- 返回值为本次合并掉的页数;通常无需手工调用(达到阈值或 VACUUM 时会自动合并)
SELECT gin_clean_pending_list('idx_name'::regclass);
--   ::regclass 的作用:把"索引名字符串"转成"索引的对象标识(OID)",该函数需要后者
```

**适用操作符**:`@>` `<@` `&&` `?` `?|` `?&` `@@`(全文)

**Recheck 机制**

GIN 的索引项可能"过宽"(例如数组的 GIN 索引会为每个元素建条目),因此查询结果可能是超集。PostgreSQL 会在 heap 层**重新校验**条件:

```
Bitmap Index Scan on gin_idx       ← 用 GIN 找候选 ctid
  Recheck Cond: (tags @> '{a,b}')
Bitmap Heap Scan on t              ← 回表
  Recheck Cond: (tags @> '{a,b}')  ← 逐行重新校验
  Rows Removed by Index Recheck: N ← 被剔除的假阳性
```

> **看到 `Rows Removed by Index Recheck` 是正常的**,但若这个数字非常大,说明索引的选择性不足(如索引了过多元素),应考虑更精确的 opclass 或部分索引。

### GiST:通用平衡树

**结构**:可扩展的平衡树,节点存放"键的边界"(如几何的包围盒)。

```
        ┌──────────────────────┐
        │  根:整个空间的包围盒  │
        └──────┬───────────────┘
        ┌──────▼──────┐  ┌──────────────┐
        │ 内部:包围盒  │  │ 内部:包围盒   │
        └──┬───────┬──┘  └──┬───────────┘
       ┌───▼──┐ ┌──▼───┐ ┌──▼───┐
       │叶子   │ │叶子   │ │叶子   │  ← 叶子项指向堆元组
       └──────┘ └──────┘ └──────┘
```

**核心特性**

| 特性 | 说明 |
|------|------|
| 键是"边界的边界" | 父节点包围所有子节点 → 支持快速剪枝 |
| 可扩展 | 通过 opclass 定义"如何合并/如何判断重叠" |
| **可能有损** | 包围盒重叠会产生假阳性 → 需 recheck |
| 支持 KNN | `ORDER BY col <-> query`(距离排序) |

**适用**:几何(PostGIS)、范围类型(`&&`)、`EXCLUDE` 约束、IP 范围。

```sql
-- 支持的操作符取决于 opclass
-- 范围类型:&&(重叠) @> <@ << >> -|-
-- 几何:&& @> <@ <->(距离) ~=
```

**`btree_gist` 的价值**:让普通类型(`text`、`int`、`timestamptz`)也能进入 GiST 索引,从而支持 `EXCLUDE` 的多列组合:

```sql
CREATE EXTENSION btree_gist;
-- 现在可以:EXCLUDE USING gist (room WITH =, during WITH &&)
--                ↑ text 列用 =(btree_gist)    ↑ 范围列用 &&(内置)
```

### SP-GiST:空间分区树

**结构**:非平衡的分区树 —— 四叉树、k-d 树、前缀树(trie)。

| 与 GiST 的差异 | 说明 |
|----------------|------|
| 树形 | 非平衡(可深可浅),而 GiST 是平衡的 |
| 分区方式 | 每次把空间切成**不相交**的子区域(减少重叠) |
| 适用 | 点数据、IP 前缀、文本前缀 |
| 不适用 | 需要平衡树保证最坏情况时 |

```sql
-- 点(四叉树)
CREATE INDEX ON points USING spgist (p);
-- inet(前缀树)
CREATE INDEX ON t USING spgist (ip);
-- 文本前缀
CREATE INDEX ON t USING spgist (name text_ops);
```

> **SP-GiST 通常也用 `Bitmap Index Scan`**,因为它的分区特性会产生较多候选。

### BRIN:块范围索引

**结构**:为"每 N 个连续数据块"记录一个摘要。

```
表的数据块(按物理顺序)
┌──────┬──────┬──────┬──────┬──────┬──────┐
│ blk0 │ blk1 │ blk2 │ blk3 │ blk4 │ blk5 │
└──────┴──────┴──────┴──────┴──────┴──────┘
    └────────┬────────┘  └────────┬────────┘
       range 0 的摘要          range 1 的摘要
       min=2026-01-01          min=2026-03-15
       max=2026-03-14          max=2026-05-30

BRIN 索引只有 2 个条目!
```

| 参数 | 说明 |
|------|------|
| `pages_per_range` | 每个 range 覆盖多少页(默认 128) |
| 摘要内容 | 取决于 opclass:`minmax`(默认)、`minmax-multi`、`inclusion`、`bloom` |

**为什么它极小**:假设 128 页/range,一个 range 摘要约 32 字节,那么 1GB 数据(131072 页)只需 1024 个条目 ≈ 32KB。

**它为什么常常没用**:BRIN 只能"排除不相关的 range",但一个 range 内仍需**逐页扫描**。有效性的唯一前提是**物理顺序与索引列强相关**。

```sql
-- 关键指标:correlation(相关性)
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'logs' AND attname = 'created_at';
-- correlation 接近 ±1 → BRIN 高效
-- correlation 接近 0  → BRIN 几乎无效
```

> **`correlation` 是整个 BRIN 决策的单点依据。** 时间序列表(只追加)通常有 0.99+ 的相关性;随机分布的表接近 0。

**BRIN vs B-tree 的量化对比**

| 维度 | BRIN | B-tree |
|------|------|--------|
| 索引大小 | 约 1/1000 | 与数据量同阶 |
| 维护成本 | 极低 | 高(每次写都维护) |
| 精确性 | 低(range 级) | 高(行级) |
| 适用前提 | 物理有序 | 无要求 |
| 典型场景 | 10 亿行日志表、IoT | 一切 |

## ③ 源码路径

| 内容 | 位置 |
|------|------|
| GIN | `src/backend/access/gin/`,结构 `src/include/access/gin*.h` |
| GiST | `src/backend/access/gist/`,结构 `src/include/access/gist*.h` |
| SP-GiST | `src/backend/access/spgist/`,结构 `src/include/access/spgist*.h` |
| BRIN | `src/backend/access/brin/`,结构 `src/include/access/brin*.h` |
| `btree_gist` | `contrib/btree_gist/` |
| `pageinspect` 的相关函数 | `contrib/pageinspect/`(`gin_metapage_info`, `gist_page_items`, `brin_page_items` 等) |

## ④ 实验验证

### 实验 1:GIN 的体积与去重

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS gin_demo;
CREATE TABLE gin_demo AS
SELECT g AS id, ARRAY(SELECT 'tag' || (g % 200 + i) FROM generate_series(1, 10) i) AS tags
FROM generate_series(1, 500000) g;

CREATE INDEX gin_demo_idx ON gin_demo USING gin (tags);
ANALYZE gin_demo;

SELECT pg_size_pretty(pg_relation_size('gin_demo'))     AS table_size,
       pg_size_pretty(pg_relation_size('gin_demo_idx')) AS gin_size;

-- 观察查询与 recheck
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM gin_demo WHERE tags @> ARRAY['tag5'];
-- 关注 Bitmap Index Scan、Recheck Cond、Rows Removed by Index Recheck
```

### 实验 2:GIN 的写入代价与 pending list

```sql
-- 观察 fastupdate 对写入的影响
DROP TABLE IF EXISTS gin_write;
CREATE TABLE gin_write (id int, tags text[]);
INSERT INTO gin_write SELECT g, ARRAY['a'||(g%1000)] FROM generate_series(1, 200000) g;

-- 建索引后批量插入
CREATE INDEX gin_write_idx ON gin_write USING gin (tags);

\timing on
INSERT INTO gin_write SELECT g, ARRAY['b'||(g%1000)] FROM generate_series(1, 200000) g;

-- 查看 pending list 情况
SELECT * FROM gin_metapage_info(get_raw_page('gin_write_idx', 0));
-- GinMetapageData 中有 nPendingPages 等字段

SELECT gin_clean_pending_list('gin_write_idx'::regclass);
```

### 实验 3:GiST 的 KNN 与 recheck

```sql
-- 需要 PostGIS 或用范围类型演示
DROP TABLE IF EXISTS gist_demo;
CREATE TABLE gist_demo (id int, r int4range);
INSERT INTO gist_demo SELECT g, int4range(g, g + 100) FROM generate_series(1, 200000) g;

CREATE INDEX gist_demo_idx ON gist_demo USING gist (r);
ANALYZE gist_demo;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM gist_demo WHERE r && int4range(5000, 5100);
-- 关注 Bitmap Index Scan + Recheck（GiST 常有损）
```

### 实验 4:BRIN 的相关性验证(最有说服力)

```sql
-- 情况 A:物理有序(时间序列表)
DROP TABLE IF EXISTS brin_ordered;
CREATE TABLE brin_ordered AS
SELECT g AS id, (timestamp '2026-01-01' + (g || ' seconds')::interval) AS ts
FROM generate_series(1, 2000000) g;

CREATE INDEX brin_ordered_idx ON brin_ordered USING brin (ts);
ANALYZE brin_ordered;

SELECT attname, correlation FROM pg_stats
WHERE tablename='brin_ordered' AND attname='ts';
-- 期望接近 1.0

SELECT pg_size_pretty(pg_relation_size('brin_ordered_idx')) AS brin_size;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM brin_ordered WHERE ts BETWEEN '2026-01-10' AND '2026-01-11';
-- 应能裁剪大部分 range

-- 情况 B:物理无序(打乱)
DROP TABLE IF EXISTS brin_shuffled;
CREATE TABLE brin_shuffled AS SELECT * FROM brin_ordered ORDER BY random();

CREATE INDEX brin_shuffled_idx ON brin_shuffled USING brin (ts);
ANALYZE brin_shuffled;

SELECT attname, correlation FROM pg_stats
WHERE tablename='brin_shuffled' AND attname='ts';
-- 期望接近 0

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM brin_shuffled WHERE ts BETWEEN '2026-01-10' AND '2026-01-11';
-- BRIN 几乎无法裁剪,退化为全扫
```

### 实验 5:对比同样数据的四种索引体积

```sql
-- 用一个统一的表测四种索引的大小
DROP TABLE IF EXISTS idx_compare;
CREATE TABLE idx_compare AS
SELECT g AS id, (g % 100000) AS num, 'v' || (g % 10000) AS txt,
       ARRAY['a' || (g % 500), 'b' || (g % 300)] AS arr
FROM generate_series(1, 500000) g;

CREATE INDEX c_btree ON idx_compare (num);
CREATE INDEX c_hash  ON idx_compare USING hash (num);
CREATE INDEX c_gin   ON idx_compare USING gin (arr);
CREATE INDEX c_brin  ON idx_compare USING brin (num);

SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes WHERE relname = 'idx_compare'
ORDER BY pg_relation_size(indexrelid);
-- 典型结果:BRIN << B-tree < Hash < GIN
```

### 实验 6:GIN 大数据量下的批量导入策略

```sql
-- ❌ 先建索引再导入
DROP TABLE IF EXISTS bulk_bad;
CREATE TABLE bulk_bad (id int, tags text[]);
CREATE INDEX bulk_bad_idx ON bulk_bad USING gin (tags);
\timing on
INSERT INTO bulk_bad SELECT g, ARRAY['x'||(g%1000)] FROM generate_series(1,500000) g;

-- ✅ 先导入再建索引
DROP TABLE IF EXISTS bulk_good;
CREATE TABLE bulk_good (id int, tags text[]);
INSERT INTO bulk_good SELECT g, ARRAY['x'||(g%1000)] FROM generate_series(1,500000) g;
CREATE INDEX bulk_good_idx ON bulk_good USING gin (tags);
-- 对比耗时:后者通常显著更快
```

## ⑤ 生产启示

| 结论 | 行动 |
|------|------|
| GIN 体积大、写慢 | 只在"多值包含查询"确实高频时用;批量导入先导数据后建索引 |
| `Rows Removed by Index Recheck` 是正常的 | 但数值极大时考虑更精确的 opclass 或部分索引 |
| GiST 有损 | 需要精确匹配时用 B-tree;GiST 是为"不精确但可剪枝"设计的 |
| `btree_gist` 是 `EXCLUDE` 的前提 | 多列排他约束必须装它 |
| BRIN 的唯一前提是物理有序 | **先看 `correlation`**,接近 0 就别用 |
| BRIN 适合超大表 | 10 亿行日志表用 BRIN 可把索引从 GB 降到 MB |
| `pages_per_range` 是权衡 | 小 → 更精确但索引更大;大 → 更小但更粗 |

```sql
-- 索引类型选择的自查
SELECT attname, correlation,
       n_distinct,
       most_common_vals[1:3] AS top_vals
FROM pg_stats
WHERE tablename = 'your_table'
ORDER BY attname;
```

```sql
-- 找出"体积大 + 使用少"的 GIN 索引(重点审查对象)
SELECT c.relname AS tbl, i.relname AS idx,
       pg_size_pretty(pg_relation_size(i.oid)) AS size,
       s.idx_scan,
       am.amname AS method
FROM pg_index x
JOIN pg_class c ON c.oid = x.indrelid
JOIN pg_class i ON i.oid = x.indexrelid
JOIN pg_am am ON am.oid = i.relam
LEFT JOIN pg_stat_user_indexes s ON s.indexrelid = i.oid
WHERE am.amname IN ('gin','gist','brin','spgist')
ORDER BY pg_relation_size(i.oid) DESC LIMIT 15;
```

## 相关页面

- [[postgresql/03-进阶阶段-深度剖析/索引内核/原理剖析-BTree结构]]
- [[postgresql/03-进阶阶段-深度剖析/索引内核/原理剖析-HOT更新与索引维护]]
- [[postgresql/02-中段阶段-专题精讲/索引实战应用/索引类型选型与设计]]
