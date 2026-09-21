---
tags: [concept, PostgreSQL, 进阶, WAL, LSN, 预写日志, 持久性, synchronous_commit]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# 原理剖析 · WAL 与 LSN 机制

## ① 现象

- 提交的数据在崩溃后不丢,但 `synchronous_commit=off` 时会丢
- 写操作产生的 WAL 量远大于数据变更量
- `pg_wal` 目录占用异常
- 只读查询也产生了 WAL
- 批量导入时磁盘 I/O 成为瓶颈且主要花在 WAL 上

## ② 机制

### 预写日志的核心规则

**WAL(Write-Ahead Logging)只有一条铁律:**

> 在把一个数据页的修改刷到磁盘**之前**,描述该修改的 WAL 记录必须**已经**持久化。

```
写流程(以 UPDATE 为例):

1. 在 shared_buffers 中修改数据页(可能是新页),标记为脏页
   └─ 页头 pd_lsn 记录本次修改的 WAL 位置
2. 把本次修改的 WAL 记录写入 WAL buffer
3. 提交时(或 WAL buffer 满时、或 walwriter 定时):
   把 WAL 刷到 WAL 文件(fsync)
4. 脏数据页留待 checkpointer/bgwriter 后续刷盘
```

**这个顺序带来的两个直接结果**

1. **提交是廉价的**:只需保证 WAL 落盘,不必把数据页写盘(随机 I/O → 顺序追加)。
2. **提交是持久的**:崩溃后可用 WAL 重放未落盘的数据页修改。

### LSN(Log Sequence Number)

**LSN 是 WAL 中的字节偏移**,64 位,表示为一个"16 进制/16 进制"形式:

```
0/1A2B3C48
│ │
│ └─ 段内偏移(低 32 位)
└─── 段号(高 32 位)
```

| LSN 语义 | 用途 |
|----------|------|
| 页头 `pd_lsn` | 该页最后被哪条 WAL 记录修改过 |
| 数据页落盘时 | 若 `pd_lsn` ≥ 记录的 LSN,说明该修改已应用(幂等,可跳过) |
| 复制进度 | `pg_stat_replication` 的 `sent_lsn` / `flush_lsn` / `replay_lsn` |
| 恢复目标 | PITR 的 `recovery_target_lsn` |

```sql
SELECT pg_current_wal_lsn();                     -- 当前写入位置
SELECT pg_current_wal_insert_lsn();              -- 当前插入位置
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0');   -- 已产生多少字节

-- 观察一次操作产生多少 WAL
SELECT pg_current_wal_lsn() AS before \gset
UPDATE orders SET status = 'done' WHERE id = 1;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'before')::bigint) AS wal_size;
```

### WAL 文件组织

| 项 | 值 |
|----|-----|
| 段大小 | 16MB(`wal_segment_size`,编译期,PG11+ 可配置但需 `initdb`) |
| 目录 | `pg_wal/`(PG10 前是 `pg_xlog/`) |
| 文件名 | `000000010000000000000001`(时间线 + 逻辑段号 + 段内段号) |

```
pg_wal/
 ├── 000000010000000000000001     ← 16MB
 ├── 000000010000000000000002
 ├── 000000010000000000000003
 ├── archive_status/              ← 归档状态标记
 └── summaries/                   ← WAL 摘要(PG17+,加速恢复)
```

**PG17+ 的 WAL 摘要**:在 WAL 中周期性插入"摘要记录",使崩溃恢复无需从头解析全部 WAL,显著缩短恢复时间。

### `wal_level` 决定记录内容

| `wal_level` | 内容 | 支持 |
|-------------|------|------|
| `minimal` | 最少记录(部分操作可跳过 WAL) | 无复制 |
| `replica`(默认) | 足够物理复制与 PITR | 流复制、PITR |
| `logical` | 额外记录逻辑解码所需信息 | 逻辑复制 |

```sql
SHOW wal_level;
-- 改 wal_level 需要重启(除 PG19 起 wal_level=replica 时可动态启用逻辑复制)
```

> **PG19 预告**:逻辑复制在新版中可在 `wal_level = replica` 下动态启用(无需重启)。这对"临时开启逻辑复制做迁移"的场景意义重大。

### 全页写(full_page_writes)

**问题**:若一页在两次检查点之间被部分写入磁盘(断电导致"页撕裂"),恢复时页内容可能是"一半新一半旧",而 WAL 记录只包含增量 → 无法正确重放。

**解法**:某页在**本次检查点周期内第一次被修改**时,WAL 记录包含**整页内容**(full-page image),而不是增量。

```sql
SHOW full_page_writes;      -- 默认 on
```

| 影响 | 说明 |
|------|------|
| 好处 | 恢复正确性(防止页撕裂) |
| 代价 | WAL 量可能大幅增加(尤其检查点间隔长时) |
| 优化 | 缩短 `checkpoint_timeout` 或增大 `max_wal_size` 可减少全页写的频率?反之 —— **检查点越频繁,每个周期内"第一次修改"越少 → 全页写总量可能减少但总 WAL 更多**。需权衡 |

### 提交与持久性

```sql
SHOW synchronous_commit;    -- on(默认)
```

| 值 | 语义 | 崩溃丢失边界 |
|----|------|--------------|
| `off` | 不等 WAL 刷盘就返回 | 可丢最近数百毫秒的事务(可能数据不一致,但**数据库不会损坏**) |
| `local` | 本地 WAL 刷盘后返回 | 本地不丢;同步备库可能丢 |
| `on`(默认) | 本地刷盘 + 同步备库写入 | 不丢 |
| `remote_write` | 备库写入 OS 缓存 | 备库 OS 崩溃可丢 |
| `remote_apply` | 备库已重放 | 最强,延迟最高 |

**关键理解:`synchronous_commit = off` 不会导致数据库损坏。**

因为 WAL 记录仍是完整有序的,只是"最后几百毫秒可能没落盘"。崩溃后数据库会恢复到某个"一致但略旧"的状态。丢失的是**已向客户端返回成功的最近若干事务**。

**组提交(group commit)**

高并发下,多个事务的 WAL 可以合并为一次 fsync:

```
时刻 T:  事务 A 请求提交
         事务 B 请求提交
         事务 C 请求提交
         ↓
         WAL writer 一次性 fsync(A+B+C 的 WAL)
         ↓
         三者同时返回成功
```

这解释了为什么"TPS 高时每次提交的成本反而降低"。

### WAL 相关参数

| 参数 | 默认 | 说明 |
|------|------|------|
| `wal_buffers` | -1(自动) | WAL 缓冲区,约 1/32 shared_buffers,上限 16MB |
| `wal_writer_delay` | 200ms | walwriter 刷 WAL 的间隔 |
| `wal_writer_flush_after` | 1MB | 达到此量立即 flush |
| `max_wal_size` | 1GB | 触发检查点的 WAL 量 |
| `min_wal_size` | 80MB | 保留的最小 WAL |
| `wal_compression` | off | 压缩 full-page image(**可显著减少 WAL 量**) |
| `commit_delay` / `commit_siblings` | 0 / 5 | 主动等待以凑成组提交 |
| `wal_keep_size` | 0 | 为复制保留的 WAL(建议改用复制槽) |

> **`wal_compression` 是常被忽略的优化**:对"含大量 full-page image 的写密集负载",开启后 WAL 量可下降 30–50%,代价是少量 CPU。PG15+ 支持 `lz4`/`zstd`。

### 与复制的联系

流复制本质上就是"把 WAL 记录传给备库并重放"。因此:

- 逻辑上**不需要单独的数据传输协议**;
- `pg_stat_replication` 中的三个 LSN(sent/write/flush/replay)就是复制进度的度量;
- 复制延迟 = `pg_current_wal_lsn()` 与 `replay_lsn` 的差值。

细节见 [[postgresql/03-进阶阶段-深度剖析/复制与高可用内核/原理剖析-流复制与同步提交]]。

## ③ 源码路径

| 内容 | 位置 |
|------|------|
| WAL 写入与刷盘 | `src/backend/access/transam/xlog.c` |
| WAL 记录构造 | `src/backend/access/transam/xloginsert.c` |
| 恢复 | `src/backend/access/transam/xlogrecovery.c` |
| WAL 缓冲区管理 | `src/backend/access/transam/xlog.c`(`XLogInsert`, `XLogFlush`) |
| walwriter 进程 | `src/backend/postmaster/walwriter.c` |
| LSN 类型与工具 | `src/include/access/xlogdefs.h` |
| `pg_walinspect` | `contrib/pg_walinspect/` |

## ④ 实验验证

### 实验 1:量化不同操作产生的 WAL

```sql
DROP TABLE IF EXISTS wal_demo;
CREATE TABLE wal_demo (id int PRIMARY KEY, v text);
INSERT INTO wal_demo SELECT g, 'v'||g FROM generate_series(1, 100000) g;
CHECKPOINT;

-- 1) INSERT
SELECT pg_current_wal_lsn() AS l \gset
INSERT INTO wal_demo SELECT g, 'v'||g FROM generate_series(200000, 300000) g;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'l')::bigint) AS wal_insert;

-- 2) UPDATE(非索引列)
SELECT pg_current_wal_lsn() AS l \gset
UPDATE wal_demo SET v = v || 'x';
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'l')::bigint) AS wal_update_nonidx;

-- 3) UPDATE(索引列)
SELECT pg_current_wal_lsn() AS l \gset
UPDATE wal_demo SET id = id + 1000000;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'l')::bigint) AS wal_update_idx;

-- 4) 只读查询
SELECT pg_current_wal_lsn() AS l \gset
SELECT count(*) FROM wal_demo;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'l')::bigint) AS wal_select;
-- 通常为 0,或极小(首次扫描设置 hint bits 时非 0)
```

### 实验 2:观察 full_page_writes 的影响

```sql
-- 强制检查点(清空"本周期内第一次修改"的记录)
CHECKPOINT;
SET full_page_writes = on;

SELECT pg_current_wal_lsn() AS l \gset
UPDATE wal_demo SET v = 'x' WHERE id <= 1000;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'l')::bigint) AS with_fpw;

-- 立刻再更新同一批行(此时不再是本周期首次修改)
SELECT pg_current_wal_lsn() AS l \gset
UPDATE wal_demo SET v = 'y' WHERE id <= 1000;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'l')::bigint) AS without_fpw;
-- 第二次显著更小(没有 full-page image)
```

### 实验 3:验证 `wal_compression` 的效果

```sql
-- 需要改配置并重启或 reload(取决于实现),此处演示思路
SHOW wal_compression;      -- off

-- 开启
ALTER SYSTEM SET wal_compression = 'lz4';
SELECT pg_reload_conf();
SHOW wal_compression;

-- 再来一次检查点后的更新,比较 WAL 量
CHECKPOINT;
SELECT pg_current_wal_lsn() AS l \gset
UPDATE wal_demo SET v = 'z' WHERE id <= 5000;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'l')::bigint) AS with_compression;
```

### 实验 4:观察 `synchronous_commit` 对吞吐的影响

```sql
-- 高延迟存储上差异明显;本地 SSD 差异较小
\timing on

SET synchronous_commit = on;
DO $$
BEGIN
    FOR i IN 1..200 LOOP
        INSERT INTO wal_demo VALUES (2000000 + i, 'x');
    END LOOP;
END $$;

SET synchronous_commit = off;
DO $$
BEGIN
    FOR i IN 1..200 LOOP
        INSERT INTO wal_demo VALUES (3000000 + i, 'x');
    END LOOP;
END $$;
```

### 实验 5:查看 WAL 记录内容

```sql
-- PG13+ 扩展方式
CREATE EXTENSION IF NOT EXISTS pg_walinspect;

SELECT * FROM pg_get_wal_records_info(
    pg_current_wal_lsn() - 1000000, pg_current_wal_lsn()
) LIMIT 10;
-- 关注 resource_manager(如 Heap)、record_type、block_refs
```

```bash
# 命令行方式(需文件访问)
pg_waldump -p /var/lib/postgresql/data/pg_wal 000000010000000000000001 | head -50
```

### 实验 6:观察 WAL 文件与回收

```sql
-- 当前 WAL 位置与文件
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());

-- WAL 目录大小(需文件系统访问)
SELECT * FROM pg_ls_waldir() ORDER BY modification LIMIT 10;
```

## ⑤ 生产启示

| 结论 | 行动 |
|------|------|
| 提交只需 WAL 落盘 | 写性能瓶颈常在 WAL,优化 WAL 往往比优化数据页更有效 |
| `wal_compression` 值得开 | 写密集 + 长检查点周期时,WAL 量可降 30–50% |
| `synchronous_commit = off` 安全但有边界 | 可用于日志/指标类场景;金融类不可用 |
| 组提交让高并发更划算 | 不要为"降低提交成本"而人为提高并发 |
| 全页写是恢复正确性的必要代价 | 不要为了省 WAL 关闭它 |
| 逻辑复制需 `wal_level = logical` | 且需重启;PG19 起 `replica` 也可动态启用 |
| WAL 目录异常膨胀 | 查复制槽、归档失败(`archive_command`)、长事务与检查点 |

```sql
-- WAL 生成速率监控
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')::bigint) AS total_wal;

-- 复制槽残留(常见 WAL 堆积原因)
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots ORDER BY 4 DESC;

-- 归档状态
SELECT * FROM pg_stat_archiver;
```

## 相关页面

- [[postgresql/03-进阶阶段-深度剖析/WAL与崩溃恢复/原理剖析-Checkpoint与崩溃恢复]]
- [[postgresql/03-进阶阶段-深度剖析/复制与高可用内核/原理剖析-流复制与同步提交]]
- [[postgresql/02-中段阶段-专题精讲/备份恢复与高可用/专题概览]]
