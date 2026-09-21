---
tags: [concept, PostgreSQL, 中段, EXPLAIN, 执行计划, 代价模型, BUFFERS]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# EXPLAIN 与 EXPLAIN ANALYZE 解读

## 一、用法与选项

```sql
EXPLAIN SELECT ...;                                  -- 只估算,不执行
EXPLAIN ANALYZE SELECT ...;                          -- 真执行,给实测
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;               -- ★日常最有用
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS, WAL) SELECT ...;
EXPLAIN (ANALYZE, FORMAT JSON) SELECT ...;           -- 机器可读
EXPLAIN (ANALYZE, COSTS OFF, TIMING OFF) SELECT ...; -- 简化输出
```

| 选项 | 作用 |
|------|------|
| `ANALYZE` | **真的执行**并收集实际统计 |
| `BUFFERS` | 显示 shared/local/temp 块的命中与读取数 |
| `VERBOSE` | 显示输出列、schema 限定名 |
| `SETTINGS` | 显示影响计划的非默认配置 |
| `WAL` | 显示生成的 WAL 量(写操作) |
| `TIMING OFF` | 关闭每节点计时(减少开销,仅看行数时用) |
| `COSTS` | 是否显示代价(默认 on) |
| `SUMMARY` | 是否显示总计(默认 on with ANALYZE) |

> **`ANALYZE` 会真正执行语句!** 对 `UPDATE`/`DELETE` 做 `EXPLAIN ANALYZE` 会真的改数据。请放在事务里并 `ROLLBACK`:
>
> ```sql
> BEGIN;
> EXPLAIN (ANALYZE, BUFFERS) UPDATE orders SET status='done' WHERE ...;
> ROLLBACK;
> ```

## 二、逐字段解读

```
Limit  (cost=0.43..12.55 rows=20 width=32) (actual time=0.021..0.089 rows=20 loops=1)
  ->  Index Scan using orders_created_desc_idx on orders  (cost=0.43..1255.21 rows=2000 width=32)
                                                          (actual time=0.019..0.083 rows=20 loops=1)
        Index Cond: (created_at > '2026-09-01'::timestamptz)
        Filter: (status = 'paid'::text)
        Rows Removed by Filter: 3
        Buffers: shared hit=24
Planning Time: 0.315 ms
Execution Time: 0.152 ms
```

| 字段 | 含义 | 看什么 |
|------|------|--------|
| `cost=起..总` | 估算代价,任意单位 | 只用于比较两个计划,不是毫秒 |
| `rows=N` | 估算输出行数 | 与实际差大 → 估算问题 |
| `width=N` | 估算行宽(字节) | 过宽说明带了大字段(如 TOAST) |
| `actual time=起..总` | 实测毫秒(含子节点) | 找耗时最大的节点 |
| `actual rows=N` | 实测输出行数 | 与 `rows` 对比 |
| `loops=N` | 节点执行次数 | 大 → 内侧被反复执行 |
| `Index Cond` | **索引层面**完成的条件 | 有它说明索引在干活 |
| `Filter` | **扫描后**再过滤的条件 | `Rows Removed by Filter` 大 → 索引不匹配 |
| `Rows Removed by Filter` | 被丢掉的行数 | 越大越浪费 |
| `Buffers: shared hit/read` | 缓存命中 / 物理读页数 | `read` 大 → 缺缓存 |
| `Heap Fetches` | Index Only Scan 回表次数 | 大 → VM 不干净,需 VACUUM |
| `Sort Method` | 排序方式 | `external merge Disk` → 落盘 |
| `Planning Time` / `Execution Time` | 规划/执行耗时 | 规划时间大于执行 → 考虑预处理语句 |

**`Index Cond` 与 `Filter` 的区别是核心**

| | 在索引里完成 | 在堆表读取后完成 |
|--|-------------|-----------------|
| 效率 | 只读需要的索引项 | 读了不该读的行再丢弃 |
| 显示为 | `Index Cond` | `Filter` + `Rows Removed by Filter` |

> 看到 `Filter` 里出现本应是主要过滤条件,说明索引设计或查询写法有问题。

## 三、估算 vs 实际:诊断统计问题

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE channel = 'web' AND status = 'paid';
```

```
Index Scan using orders_channel_idx on orders
  (cost=0.43..180.20 rows=120 width=64) (actual time=0.03..142.5 rows=45210 loops=1)
  Index Cond: (channel = 'web')
  Filter: (status = 'paid')
  Rows Removed by Filter: 118000
  Buffers: shared hit=98234 read=4211
```

**两个问题同时暴露**:

1. 估算 `rows=120` vs 实际 `45210` → 差 377 倍。
2. `Rows Removed by Filter: 118000` → 读了 16 万行只为留 4.5 万。

**对策**

```sql
-- 1) 更新统计
ANALYZE orders;

-- 2) 若 channel 与 status 强相关,建多变量统计
CREATE STATISTICS orders_ch_stat (dependencies, ndistinct) ON channel, status FROM orders;
ANALYZE orders;

-- 3) 提高采样精度
ALTER TABLE orders ALTER COLUMN channel SET STATISTICS 1000;
ANALYZE orders;

-- 4) 优化索引:把 status 也放进索引键(或改部分索引)
CREATE INDEX ON orders (channel, status);
-- 或
CREATE INDEX ON orders (channel) WHERE status = 'paid';
```

## 四、Buffers 解读

```sql
SET track_io_timing = on;      -- 让 BUFFERS 显示真实 I/O 时间
EXPLAIN (ANALYZE, BUFFERS) ...;
```

```
Buffers: shared hit=1250 read=312 dirtied=12 written=5,
         local hit=0 read=0,
         temp read=800 written=800
```

| 字段 | 含义 |
|------|------|
| `shared hit` | 命中共享缓冲区的页数(快) |
| `shared read` | 从磁盘(或 OS 缓存)读入的页数,每页 8KB |
| `shared dirtied` | 被弄脏的页 |
| `shared written` | 被写盘的页 |
| `local` | 临时表占用 |
| `temp` | 临时文件(排序/哈希落盘),**`temp written` > 0 说明内存不足** |

**换算**:`shared read=312` ≈ 312 × 8KB ≈ 2.5MB 物理读。

**缓存命中率**:

```
命中率 = hit / (hit + read)
```

理想值 > 99%。低于 90% 说明工作集超出 `shared_buffers`,或访问模式随机。

## 五、常见计划的"病因对照表"

| 计划特征 | 病因 | 对策 |
|----------|------|------|
| `Seq Scan` + `Rows Removed by Filter` 大 | 缺索引 / 索引不可用 | 建索引;检查函数、类型转换、部分索引条件 |
| `Seq Scan` 且表很小 | 正常(小表全扫更快) | 不动 |
| `Sort Method: external merge Disk` | `work_mem` 不足 | 调大 `work_mem`;建同序索引;减少行数 |
| `Hash` 节点 `Batches > 1` | 哈希表超出 `work_mem`,分批 | 调大 `work_mem` |
| `rows` 估算 ≪ 实际 | 低估 → 可能误选 Nested Loop | `ANALYZE`;多变量统计 |
| `rows` 估算 ≫ 实际 | 高估 → 可能误选 Hash/Merge Join | 同上 |
| `Nested Loop` + `loops` 大 | 内侧反复扫描 | 内侧连接列建索引 |
| `Heap Fetches` 大 | VM 不干净 | `VACUUM` |
| `Buffers: read` 大 | 缓存未命中 | 增大 `shared_buffers`;减少扫描量 |
| `temp written` 大 | 排序/哈希落盘 | 调大 `work_mem` |
| 计划波动大(同 SQL 不同计划) | 通用计划 vs 定制计划 | 视情况用 `PREPARE`/`plan_cache_mode` |

## 六、代价模型参数(理解规划器偏好)

```sql
SHOW seq_page_cost;        -- 默认 1.0
SHOW random_page_cost;     -- 默认 4.0(SSD 建议 1.1~2.0)
SHOW cpu_tuple_cost;       -- 默认 0.01
SHOW cpu_index_tuple_cost; -- 默认 0.005
SHOW cpu_operator_cost;    -- 默认 0.0025
SHOW effective_cache_size; -- 默认 4GB
SHOW parallel_setup_cost;  -- 默认 1000
```

> **SSD 上必调 `random_page_cost`**:默认 `4.0` 反映的是机械盘"随机读比顺序读贵 4 倍"的假设。SSD 上没有这个差异,保持 4.0 会让规划器**过度偏好 Seq Scan**(因为认为走索引的随机读太贵)。降到 `1.1~2.0` 能让它更愿意用索引。

```sql
-- 单条查询临时调参,验证是否是代价模型问题
SET random_page_cost = 1.1;
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
RESET random_page_cost;
```

## 七、计划缓存与通用计划

预处理语句(以及某些驱动)会使用通用计划(generic plan),可能因不感知参数值而选错计划。

```sql
PREPARE p(int) AS SELECT * FROM orders WHERE user_id = $1;

-- 观察是否用通用计划
EXPLAIN EXECUTE p(123);

SHOW plan_cache_mode;      -- auto / force_generic_plan / force_custom_plan
-- auto:前 5 次用定制计划,之后视情况切换
```

**症状**:应用运行时慢,手工用具体参数跑却很快。对策:

```sql
SET plan_cache_mode = force_custom_plan;   -- 每参数定制计划(代价:规划开销)
```

## 八、auto_explain:自动记录慢查询计划

```sql
-- postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '1s'
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_nested_statements = on
```

重启后生效。用于抓"生产里偶发的慢查询",此时你无法手工跑 `EXPLAIN`。

## 九、实战对比模板

```sql
-- 优化前后各跑一次,记录关键指标
\timing on
EXPLAIN (ANALYZE, BUFFERS) <原始 SQL>;
-- 记录:Execution Time、Buffers: read、Rows Removed by Filter

CREATE INDEX CONCURRENTLY ... ;
ANALYZE <table>;

EXPLAIN (ANALYZE, BUFFERS) <原始 SQL>;
-- 对比:Execution Time 是否下降、Buffers 是否下降、计划是否改变
```

**验证有效性的标准**:不能只看 `Execution Time`(受缓存影响大),要同时看 **Buffers**。若 `Execution Time` 降了但 `Buffers` 没降,可能只是缓存变热了。

## 相关页面

- [[postgresql/02-中段阶段-专题精讲/执行计划与性能诊断/专题概览]]
- [[postgresql/02-中段阶段-专题精讲/索引实战应用/实战-慢查询索引优化]]
- [[postgresql/03-进阶阶段-深度剖析/查询优化器/原理剖析-统计信息与基数估计]]
