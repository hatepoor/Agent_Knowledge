---
tags: [concept, PostgreSQL, 进阶, TOAST, 行版本, 大字段, 压缩]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# 原理剖析 · TOAST 与行版本

## ① 现象

- 一张表只有几行,但 `pg_total_relation_size` 远大于 `pg_relation_size`
- 存了大 `text` 后,查询其他列的速度也变慢
- `UPDATE` 长文本字段后,表与索引膨胀尤其严重
- 行头长度与"所有列长度之和"对不上
- 压缩阈值行为不符合预期

## ② 机制

### 为什么需要 TOAST

一个 8KB 页要放多行,而 PostgreSQL **不允许单个元组跨页**(这是设计约束,简化了并发与访问)。

若一行数据超过约 2KB,就必须想办法"缩短"它。TOAST 就是这套机制:

**TOAST = The Oversized-Attribute Storage Technique(超尺寸属性存储技术)**

### 触发阈值

```
TOAST_TUPLE_THRESHOLD = 2KB       ← 超过则触发 TOAST 处理
TOAST_TUPLE_TARGET    = 2KB       ← “压缩/外置”后希望达到的行大小(可调)
```

| 参数 | 默认 | 说明 |
|------|------|------|
| `toast_tuple_target` | 2KB | 表级参数,目标元组大小 |
| `TOAST_MAX_CHUNK_SIZE` | 约 2KB | 每个 TOAST 块的大小 |

```sql
-- 表级调整
ALTER TABLE docs SET (toast_tuple_target = 4096);
```

### 四级存储策略(每列可单独指定)

| 策略 | 含义 | 是否压缩 | 是否外置 |
|------|------|----------|----------|
| `PLAIN` | 不做任何处理(仅定长类型) | ❌ | ❌ |
| `EXTENDED`(默认) | 先压缩,仍超限则外置 | ✅ | ✅ |
| `EXTERNAL` | 不压缩,直接外置 | ❌ | ✅ |
| `MAIN` | 优先压缩,尽量不外置 | ✅ | 尽量不 |

```sql
-- 查看列的存储策略
SELECT attname, attstorage
FROM pg_attribute WHERE attrelid = 'docs'::regclass AND attnum > 0;
-- p=PLAIN, e=EXTENDED, x=EXTERNAL, m=MAIN

-- 修改
ALTER TABLE docs ALTER COLUMN body SET STORAGE EXTERNAL;
```

**策略选择的实践意义**

| 场景 | 选择 | 理由 |
|------|------|------|
| 大文本,常整块返回 | `EXTENDED`(默认) | 压缩省空间 |
| 已压缩数据(图片、zip) | `EXTERNAL` | 再压无效,浪费 CPU |
| 需子串访问的文本 | `EXTERNAL` | 外置后可按需读取部分数据,不必解压整块 |
| 小字段 | `PLAIN` 或默认 | — |

### 处理流程

```
行总长 > 2KB?
   │
   ├─ 否 → 正常存入该页
   │
   └─ 是 → 按列依次尝试(从最大的变长列开始)
            │
            ├─ 策略含压缩 → 压缩(若压缩有效则保留压缩结果)
            │
            └─ 仍超 2KB → 外置(out-of-line)
                         整列数据写成 TOAST 表中的多个 chunk,
                         原行只保留一个 18 字节的 TOAST 指针
```

**压缩算法(PG14+)**

| 算法 | 参数值 | 说明 |
|------|--------|------|
| `pglz` | `pglz` | 传统算法,较慢,压缩率中等 |
| `lz4` | `lz4` | **PG14+**,更快,压缩率相近 |

```sql
-- 默认压缩方法(实例级)
SHOW default_toast_compression;      -- 默认 pglz
-- 列级指定
ALTER TABLE docs ALTER COLUMN body SET COMPRESSION lz4;
```

> **需要编译时支持 lz4**(`--with-lz4`)。若不可用,指定 `lz4` 会报错。

### TOAST 表与索引

| 对象 | 说明 |
|------|------|
| `pg_toast.pg_toast_<oid>` | 主表对应的 TOAST 表,存放外置的 chunk |
| `pg_toast_<oid>_index` | TOAST 表上的 B-tree 索引(`chunk_id`, `chunk_seq`) |

```sql
-- 查看某表的 TOAST 表
SELECT c.relname AS main_table,
       t.relname AS toast_table,
       pg_size_pretty(pg_relation_size(t.oid)) AS toast_size
FROM pg_class c
JOIN pg_class t ON t.oid = c.reltoastrelid
WHERE c.relname = 'docs';
```

> **重要**:TOAST 表**有索引**。这意味着更新大字段时,不仅要写主表页,还要写 TOAST 表的页与索引页 —— 写入放大显著。这是"更新长文本很贵"的量化解释。

### 行版本(tuple version)回顾

"行版本"是 MVCC 的实体:每次 `UPDATE` 产生一个新元组版本,旧版本被标记为不可见。

| 概念 | 说明 |
|------|------|
| 元组版本 | (块号, 槽号) 唯一定位,即 `ctid` |
| 版本链 | 通过 `t_ctid` 串联(用于 HOT 更新) |
| 死元组 | 对所有活跃快照都不可见的旧版本 |
| 冻结 | 把老元组的 `xmin` 标记为 `FrozenTransactionId`,使 xid 回绕不影响可见性 |

**空间放大**

```
逻辑行数 100 万
平均每行被更新 5 次
→ 磁盘上曾有约 600 万个元组版本
→ VACUUM 前,表文件按 600 万行占位
→ VACUUM 后空间可复用,但文件不缩小
```

## ③ 源码路径

| 内容 | 位置 |
|------|------|
| TOAST 主逻辑 | `src/backend/access/heap/tuptoaster.c` |
| 压缩实现 | `src/backend/access/common/toast_compression.c` |
| pglz | `src/backend/utils/adt/pg_lzcompress.c` |
| 元组结构(TOAST 指针) | `src/include/access/htup_details.h`(`varattrib_1b_e`) |
| TOAST 表创建 | `src/backend/catalog/toasting.c` |
| TOAST 相关参数 | `src/include/access/toast_internals.h` |

## ④ 实验验证

### 实验 1:观察 TOAST 触发的空间分层

```sql
DROP TABLE IF EXISTS toast_demo;
CREATE TABLE toast_demo (
    id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    small text,
    big   text
);

-- 插入较短的数据(不触发 TOAST)
INSERT INTO toast_demo (small, big)
SELECT 'x', repeat('y', 500) FROM generate_series(1, 1000);
VACUUM ANALYZE;
SELECT pg_size_pretty(pg_relation_size('toast_demo'))       AS main,
       pg_size_pretty(pg_total_relation_size('toast_demo')) AS total;

-- 插入很长的数据(触发 TOAST)
DELETE FROM toast_demo;
INSERT INTO toast_demo (small, big)
SELECT 'x', repeat('y', 50000) FROM generate_series(1, 1000);
VACUUM ANALYZE;
SELECT pg_size_pretty(pg_relation_size('toast_demo'))       AS main,
       pg_size_pretty(pg_total_relation_size('toast_demo')) AS total;
-- main 变化很小(存放 TOAST 指针),total 大幅增长(数据在 TOAST 表)
```

### 实验 2:看到 TOAST 指针

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp, lp_len, t_hoff
FROM heap_page_items(get_raw_page('toast_demo', 0))
LIMIT 5;
-- 注意 lp_len:虽然 big 有 5 万字节,但元组长度很小(只有指针)
```

### 实验 3:验证压缩效果

```sql
DROP TABLE IF EXISTS comp_demo;
CREATE TABLE comp_demo (id int, body text);

-- 高压缩率数据(重复文本)
INSERT INTO comp_demo
SELECT g, repeat('abcdefghij', 5000) FROM generate_series(1, 200);
VACUUM ANALYZE;
SELECT 'compressible' AS kind, pg_size_pretty(pg_total_relation_size('comp_demo'));

-- 不可压缩数据(随机)
DELETE FROM comp_demo;
INSERT INTO comp_demo
SELECT g, (SELECT string_agg(md5(random()::text), '') FROM generate_series(1, 156))
FROM generate_series(1, 200);
VACUUM ANALYZE;
SELECT 'random' AS kind, pg_size_pretty(pg_total_relation_size('comp_demo'));
```

### 实验 4:对比 lz4 与 pglz

```sql
ALTER TABLE comp_demo ALTER COLUMN body SET COMPRESSION lz4;
-- 需重建表才能应用新压缩方式
CREATE TABLE comp_lz4 (LIKE comp_demo);
ALTER TABLE comp_lz4 ALTER COLUMN body SET COMPRESSION lz4;
INSERT INTO comp_lz4 SELECT * FROM comp_demo;
VACUUM ANALYZE;
SELECT pg_size_pretty(pg_total_relation_size('comp_lz4')) AS lz4,
       pg_size_pretty(pg_total_relation_size('comp_demo')) AS pglz;
```

### 实验 5:验证 EXTERNAL 策略跳过压缩

```sql
CREATE TABLE ext_demo (id int, body text);
ALTER TABLE ext_demo ALTER COLUMN body SET STORAGE EXTERNAL;

-- 插入可压缩数据
INSERT INTO ext_demo SELECT g, repeat('a', 100000) FROM generate_series(1, 100);
VACUUM ANALYZE;
SELECT pg_size_pretty(pg_total_relation_size('ext_demo'));
-- 明显大于使用 EXTENDED 的同数据表(因为没压缩)

-- 但取子串可能更快(无需解压整块)
SELECT substring(body from 1 for 10) FROM ext_demo LIMIT 1;
```

### 实验 6:更新大字段的写入放大

```sql
RESET work_mem;
SELECT pg_current_wal_lsn() AS lsn0;   \gset

UPDATE comp_demo SET body = body || 'x' WHERE id <= 100;

SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'lsn0')::bigint) AS wal_generated;
-- 对比更新一个小字段产生的 WAL 量,可见写入放大
```

## ⑤ 生产启示

| 结论 | 行动 |
|------|------|
| 大字段走 TOAST,且**有索引** | 更新大字段的代价远高于更新小字段;避免频繁更新大文本 |
| 已压缩数据应设 `EXTERNAL` | 图片/zip/加密数据再压缩无收益,徒耗 CPU |
| 需要子串访问设置 `EXTERNAL` | 避免解压整块 |
| `SELECT *` 会触发 TOAST 读 | **只取需要的列**是真实优化,不是教条 |
| `pg_total_relation_size` 才看全貌 | 评估表大小必须用 total(含 TOAST + 索引) |
| 极端大字段考虑外置存储 | 单值超过几十 KB 时,对象存储 + 存引用往往更优(但失去事务一致性,需权衡) |
| `lz4` 优先于 `pglz` | 若编译支持,`lz4` 在现代硬件上更快且压缩率接近 |
| TOAST 表也需 VACUUM | 大字段更新产生的死元组在 TOAST 表里,同样需要清理 |

```sql
-- 找出"主表小但总量大"的表(TOAST 大户)
SELECT c.relname,
       pg_size_pretty(pg_relation_size(c.oid))        AS main_size,
       pg_size_pretty(pg_relation_size(c.reltoastrelid)) AS toast_size,
       pg_size_pretty(pg_total_relation_size(c.oid))  AS total
FROM pg_class c
WHERE c.relkind = 'r' AND c.reltoastrelid <> 0
ORDER BY pg_relation_size(c.reltoastrelid) DESC
LIMIT 10;
```

## 相关页面

- [[postgresql/03-进阶阶段-深度剖析/存储引擎与页结构/原理剖析-堆表与页结构]]
- [[postgresql/03-进阶阶段-深度剖析/存储引擎与页结构/实验验证-物理布局观测]]
- [[postgresql/03-进阶阶段-深度剖析/事务与MVCC内核/原理剖析-MVCC与快照隔离]]
