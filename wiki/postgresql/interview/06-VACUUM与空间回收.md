---
tags: [interview, postgresql, vacuum, autovacuum, 膨胀, 面试]
type: guide
created: 2026-10-18
updated: 2026-10-18
---

# VACUUM 与空间回收

> **适用场景**：PG 面试的「深度题」。答得出 VACUUM，说明懂 MVCC 的代价；答不出，说明只停留在「加索引」。
> **主线**：MVCC 产生死元组 → VACUUM 回收 → 回收不及时就膨胀 → 膨胀拖垮性能。

---

## Q1. 为什么 PostgreSQL 需要 VACUUM？

**答：**

因为 MVCC 的代价是**产生死元组（dead tuple）**。

```sql
UPDATE users SET name = '新名字' WHERE id = 1;
```

实际发生的是：

```
旧行：xmin=100, xmax=200   ← 被标记删除，但物理上还在
新行：xmin=200, xmax=0     ← 新版本
```

旧行不会立即消失，因为**可能还有老事务的快照需要看到它**。等所有需要它的事务结束后，它就成了「死元组」，必须由 VACUUM 清理。

VACUUM 做四件事：

| 作用 | 说明 |
|------|------|
| **回收死元组占用的空间** | 标记为可复用（不是立即还给 OS） |
| **更新 visibility map** | 标记「全可见页」，让 Index Only Scan 不用回表 |
| **冻结老事务 ID** | 防止 XID 回卷 |
| **更新统计信息**（VACUUM ANALYZE） | 让优化器估算更准 |

**不 VACUUM 的后果**：表/索引膨胀 → 扫描页数增加 → 缓存命中下降 → 查询变慢；最终 XID 老化，触发停库保护。

**追问：DELETE 之后磁盘空间会变小吗？**
→ 不会（除非删的是文件末尾的页）。空间只在表文件内部标记为可复用，**文件大小不变**。要让文件变小，需要 `VACUUM FULL` 或 `pg_repack`。

---

## Q2. `VACUUM`、`VACUUM FULL`、`VACUUM ANALYZE` 有什么区别？

**答：**

| 命令 | 做什么 | 锁 | 返还 OS 空间 | 适用 |
|------|--------|----|-------------|------|
| **`VACUUM`** | 清理死元组，标记空间可复用 | `SHARE UPDATE EXCLUSIVE`（不阻塞读写） | ❌（仅末尾页可截断） | 日常 |
| **`VACUUM FULL`** | **重写整张表**，紧凑排列 | **`ACCESS EXCLUSIVE`（阻塞一切）** | ✅ | 严重膨胀、停机窗口 |
| **`VACUUM ANALYZE`** | 清理 + 收集统计信息 | 同上（轻量） | ❌ | 大批量变更后 |
| **`VACUUM (FREEZE)`** | 强制冻结老事务 ID | 轻量 | ❌ | XID 老化告警时 |

⚠️ **`VACUUM FULL` 的三个代价**：

1. **全程锁表**，业务停摆；
2. **需要额外磁盘空间**（重写期间新旧表并存，约为表大小的 1 倍）；
3. **会产生大量 WAL**，复制延迟飙升。

**生产替代方案**：`pg_repack` 扩展——在线重组织，最后只短暂持锁切换。

**追问：`VACUUM` 会不会阻塞查询？**
→ 普通 VACUUM 不阻塞读写，但会**争用 I/O 和 CPU**；它需要 `SHARE UPDATE EXCLUSIVE` 锁，会与 DDL、其他 VACUUM 冲突。大表 VACUUM 期间查询变慢是常见现象。

---

## Q3. autovacuum 是怎么工作的？关键参数有哪些？

**答：**

autovacuum 是**后台自动清理机制**：一个 launcher 进程 + 多个 worker 进程。

触发逻辑（简化）：

```
死元组数 > autovacuum_vacuum_threshold
          + autovacuum_vacuum_scale_factor × 表行数
→ 触发 VACUUM
```

关键参数：

| 参数 | 默认 | 说明 |
|------|------|------|
| `autovacuum` | on | **生产绝不能关** |
| `autovacuum_vacuum_threshold` | 50 | 死元组基数阈值 |
| `autovacuum_vacuum_scale_factor` | 0.2 | 按表大小比例触发（**大表的坑**） |
| `autovacuum_analyze_threshold` | 50 | 统计信息更新阈值 |
| `autovacuum_analyze_scale_factor` | 0.1 | 同上 |
| `autovacuum_naptime` | 1min | 巡检间隔 |
| `autovacuum_max_workers` | 3 | 并发 worker 数 |
| `autovacuum_vacuum_cost_limit` | -1（取 vacuum_cost_limit） | 每轮 I/O 预算 |
| `autovacuum_vacuum_cost_delay` | 2ms（新版默认可能不同） | 超预算后的休眠 |

**大表的问题**：`scale_factor = 0.2` 意味着**1 亿行的表要产生 2000 万死元组才触发**。所以大表要单独调低：

```sql
ALTER TABLE big_table SET (autovacuum_vacuum_scale_factor = 0.01);
```

**追问：autovacuum 会不会影响业务？**
→ 会。它和业务争 I/O。调优方向是**限速而不是关掉**：用 `cost_limit`/`cost_delay` 控制节奏，或在低峰期手动 VACUUM。

---

## Q4. 怎么判断表膨胀了？膨胀了怎么处理？

**答：**

**判断三步**：

```sql
-- ① 看死元组和最后清理时间
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, last_vacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- ② 看表/索引的实际大小
SELECT pg_size_pretty(pg_total_relation_size('orders'));

-- ③ 精确测量（需要 pgstattuple 扩展）
SELECT * FROM pgstattuple('orders');
```

膨胀的典型信号：

- 表大小持续增长，但行数稳定；
- `n_dead_tup` 长期很高；
- 同样的查询扫描页数越来越多；
- 索引也膨胀（`pgstatindex` 看 avg_leaf_density）。

**处理手段**：

| 手段 | 适用 | 代价 |
|------|------|------|
| **调 autovacuum 参数** | 预防/轻中度 | 无（后台） |
| **手动 `VACUUM`** | 死元组多但空间可复用 | 轻 |
| **`VACUUM FULL`** | 严重膨胀、可停机 | 锁表 + 磁盘 |
| **`pg_repack`** | 严重膨胀、不能停机 | 需要额外空间，最后短暂锁 |
| **分区表 + DROP PARTITION** | 按时间淘汰数据 | 最彻底，设计阶段就要做 |

**追问：为什么「删了很多数据」之后表反而更慢？**
→ 删除产生死元组，表文件没变小，扫描仍要读这些页；索引也没清理。**删除后的第一步应该是 VACUUM（或 VACUUM FULL），不是继续加索引。**

---

## Q5. 为什么普通 VACUUM 不把磁盘空间还给操作系统？

**答：**

因为 PG 的空间复用是**表文件内部**的：

```
表文件
┌────┬────┬────┬────┬────┐
│页1 │页2 │页3 │页4 │页5 │
│活  │死  │活  │死  │活  │
└────┴────┴────┴────┴────┘
        ↑ VACUUM 后标记为空闲，可被新行复用
```

- VACUUM 把死元组占用的空间标记为 **free space**，后续 INSERT/UPDATE 可以复用；
- 但**文件大小不变**，因为中间的空闲页无法直接删除（后面的页还要用）；
- 只有**文件末尾的连续空页**可以被截断（truncate）还给 OS。

为什么不直接搬移页面来压缩？

- 搬移页面要更新**所有索引里的 `ctid`**，代价极高；
- 还要持有强锁，阻塞业务；
- 所以 PG 选择「标记复用」这种低成本方案，把「真正压缩」交给 `VACUUM FULL`/`pg_repack`。

**追问：那为什么还要担心膨胀？**
→ 因为**扫描是按页进行的**。表里有 50% 死空间，一次全表扫描就要多读一倍的页；缓存里也有一半是无效数据。**空间不还回去不是问题，「有效数据密度下降」才是问题。**

---

## Q6. freeze 是什么？它和事务 ID 回卷是什么关系？

**答：**

freeze 是**把行版本标记为「永久可见」**，不再依赖 `xmin` 判断。

背景：

- 事务 ID 是 32 位，约 42 亿个，会循环；
- 循环后，「很老的事务」和「很新的事务」在数值上无法区分；
- 如果不处理，MVCC 的可见性判断会错乱。

freeze 的做法：

```
普通行：xmin = 100（要比较事务状态）
冻结行：xmin = FrozenXID（特殊值，永远视为已提交且可见）
```

触发：

| 机制 | 说明 |
|------|------|
| **VACUUM** | 清理时顺带冻结「足够老」的行 |
| **VACUUM FREEZE** | 强制冻结 |
| **autovacuum** | 自动触发，正常无需干预 |

监控：

```sql
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;
```

`age` 接近 `autovacuum_freeze_max_age` 就要处理；接近 20 亿，PG 会拒绝写入以保护数据。

**追问：为什么「不做 VACUUM 会停库」？**
→ 不是磁盘满，是 **XID 耗尽**。PG 无法安全判断行版本的新旧，只能进入只读/停机状态，强制 DBA 处理。这是保护机制，不是故障。

---

## Q7. 什么情况会阻止 VACUUM 回收死元组？

**答：**

VACUUM 只能清理「**所有可能用到它的快照都已经不存在**」的死元组。以下情况会让它「看得见但不敢删」：

| 阻止因素 | 机制 |
|---------|------|
| **长事务** | 持有老快照，死元组对它仍可见 |
| **长 SQL 查询** | 查询期间快照一直有效，哪怕没开显式事务 |
| **未提交的 prepared transaction** | 两阶段提交悬挂，快照一直挂着 |
| **复制槽（replication slot）** | 从库还没消费的 WAL 对应的旧版本不能清 |
| **逻辑复制订阅** | 类似的保留机制 |
| **备份/只读事务** | `pg_dump`、`BEGIN ISOLATION LEVEL REPEATABLE READ` 长时间持有快照 |
| **autovacuum 被禁用/worker 不足** | 根本没人来清理 |

**排查**：

```sql
-- 谁持有最老的快照
SELECT pid, backend_xmin, state, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

**追问：为什么「一个长事务」能让整库膨胀？**
→ 因为 VACUUM 的回收边界是**全局最老快照**。一个事务从早上开到现在，所有表在这期间产生的死元组都清不掉。**长事务是全局性问题，不是单表问题。**

---

## Q8. 怎么监控 VACUUM 的运行状态和效果？

**答：**

三个层面：

**① 实时进度（PG 9.6+）**

```sql
SELECT pid, relid::regclass, phase,
       heap_blks_total, heap_blks_scanned, heap_blks_vacuumed,
       index_vacuum_count, max_dead_tuples, num_dead_tuples
FROM pg_stat_progress_vacuum;
```

`phase` 能看到「扫描堆 / 清理索引 / 清理堆」等阶段。

**② 历史与健康度**

```sql
SELECT relname, n_dead_tup, n_live_tup,
       last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

**③ 关键告警项**

| 指标 | 阈值建议 |
|------|---------|
| `n_dead_tup / n_live_tup` | 持续 > 20% 告警 |
| `last_autovacuum` | 高写入表超过数小时未清理 |
| `age(datfrozenxid)` | 超过 `autovacuum_freeze_max_age` 的 50% 预警 |
| `pg_stat_progress_vacuum` 长时间停留 | 可能被长事务/锁阻塞 |
| 临时文件/IO 突增 | VACUUM 限速不合理 |

**追问：为什么 VACUUM 跑了很久还没结束？**
→ 常见：① 表太大且 `cost_delay` 限速；② 被锁等待卡住；③ 索引太多，清理索引阶段慢；④ 同时有长事务，导致反复清理同一批死元组。**看 `pg_stat_progress_vacuum` 的 phase 和 `pg_locks` 能定位。**

---

## 本篇自测

- [ ] MVCC 为什么产生死元组，VACUUM 做的四件事
- [ ] VACUUM / VACUUM FULL / VACUUM ANALYZE 的锁与空间差异
- [ ] autovacuum 的触发公式，大表为什么要单独调 scale_factor
- [ ] 判断膨胀的三步，四种处理手段
- [ ] 为什么普通 VACUUM 不还空间给 OS
- [ ] freeze 的原理，XID 耗尽为什么停库
- [ ] 七种阻止 VACUUM 的因素
- [ ] 三个层面的监控手段

---

## 导航

[[postgresql/interview/README|PostgreSQL 面试题库首页]] · 上一篇：[[postgresql/interview/05-查询优化与执行计划]] · 下一篇：[[postgresql/interview/07-复制备份与高可用]] · 详解：[[postgresql/03-进阶阶段-深度剖析/存储引擎与页结构/原理剖析-TOAST与行版本]]
