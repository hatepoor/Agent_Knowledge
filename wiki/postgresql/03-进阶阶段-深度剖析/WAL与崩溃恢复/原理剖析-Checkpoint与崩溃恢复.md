---
tags: [concept, PostgreSQL, 进阶, Checkpoint, 崩溃恢复, 控制文件, 恢复]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# 原理剖析 · Checkpoint 与崩溃恢复

## ① 现象

- 崩溃重启后恢复时间很长(几分钟到几十分钟)
- 日志里反复出现 `checkpoints are occurring too frequently`
- 周期性 I/O 尖峰(检查点导致)
- `pg_wal` 目录突然增长
- 恢复时看到 "redo starts at ..." 与 "redo done at ..."

## ② 机制

### 检查点要解决什么

**问题**:若从不把脏页刷盘,崩溃后需重放**全部历史 WAL** —— 恢复时间不可控。

**检查点**:把某个时刻之前的所有脏页刷盘,并记录"从这里开始的 WAL 足以恢复"。

```
WAL 时间轴:
─────────────┬────────────────────────────┬───────────►
             │                            │
        REDO 起点                    检查点完成
        (redo point)                (checkpoint LSN)
             │                            │
     崩溃后从这里重放 ────────────────────►│
     (之前的修改都已落盘,无需重放)
```

**核心概念对**

| 概念 | 含义 |
|------|------|
| `REDO` 起点(redo point) | 检查点**开始时**的 WAL 位置 —— 恢复从这里重放 |
| 检查点完成位置 | 检查点**结束时**的 WAL 位置 |
| 恢复范围 | `[redo point, 结束]` |

> **为什么 REDO 是"开始"而不是"结束"?** 因为检查点期间仍在写脏页与产生 WAL。为保证不漏,必须从开始时重放;重放时通过页头 `pd_lsn` 判断"该修改是否已应用",从而保证幂等。

### 两类检查点

| 类型 | 触发方式 | 特点 |
|------|----------|------|
| **定时检查点** | `checkpoint_timeout`(默认 5min) | 平滑、可控 |
| **需求检查点** | WAL 量达 `max_wal_size` | **强制、紧急**;日志会警告 "occurring too frequently" |

**检查点的三个参数**

| 参数 | 默认 | 作用 |
|------|------|------|
| `checkpoint_timeout` | 5min | 最长间隔 |
| `max_wal_size` | 1GB | 触发检查点的 WAL 上限 |
| `checkpoint_completion_target` | 0.9 | 在间隔的 90% 时间内**摊平**刷盘 I/O |
| `checkpoint_flush_after` | 256kB | 刷多少字节后主动让出 |
| `checkpoint_warning` | 30s | 检查点间隔短于此值则告警 |

**`checkpoint_completion_target = 0.9` 的意义**

```
若 checkpoint_timeout = 5min,completion_target = 0.9
→ 刷盘应分布在 4.5 分钟内完成,而非瞬间爆发
→ 避免 I/O 尖峰影响正常查询

若设得太小(如 0.1):
→ 检查点在工作周期的 10% 内完成,大量 I/O 堆积在短时间
→ 出现周期性性能抖动
```

### 脏页刷盘的两个角色

| 进程 | 行为 |
|------|------|
| `bgwriter` | 持续、小批量刷脏页,保持缓冲池有空闲页 |
| `checkpointer` | 按上述策略推进检查点 |

**注意**:`checkpointer` 才是"检查点的负责人";`bgwriter` 只是帮忙平滑 I/O。两者都通过 WAL 保证一致性。

### 崩溃恢复的完整流程

```
① postmaster 检测到异常(无 clean shutdown 标记)
        ↓
② 读取控制文件 pg_control
   → 得到 checkpoint 位置(redo point)、状态、时间线
        ↓
③ 进入恢复模式,启动 startup 进程
        ↓
④ 从 redo point 开始顺序重放 WAL
   → 对每条记录:若目标页 pd_lsn >= 记录 LSN,跳过(幂等)
        ↓
⑤ 重放到 WAL 末尾(或达到恢复目标)
        ↓
⑥ 写一个检查点,标记实例状态为正常
        ↓
⑦ 接受连接(主库场景)
```

**PG17+ 的 WAL 摘要加速**:WAL 中周期性插入摘要记录,使恢复无需解析全部记录即可定位必要信息,**显著缩短恢复时间**。

### 控制文件(pg_control)

`pg_controldata` 可读,包含:

| 字段 | 含义 |
|------|------|
| `pg_control version number` | 数据目录格式版本 |
| `Database cluster state` | `shut down` / `in production` / `in archive recovery` 等 |
| `Latest checkpoint location` | 最近检查点位置 |
| `Prior checkpoint location` | 上一个检查点 |
| `Latest checkpoint's REDO location` | **恢复起点** |
| `Latest checkpoint's TimeLineID` | 时间线 |
| `Latest checkpoint's NextXID` | 下一个事务 ID |
| `Latest checkpoint's NextOID` | 下一个对象 ID |
| `Checksum version` | 页校验和版本 |

```bash
pg_controldata /var/lib/postgresql/data
```

> **`Database cluster state` 是关键诊断信息**:若显示 `in production` 而实例未启动,说明上次是**崩溃退出**;正常运行时会显示 `in production`。

### 时间线(timeline)

时间线用于区分"同一历史的不同分支"(PITR 后产生新时间线)。

```
        ┌─ timeline 1(原始)─┬─ timeline 2(PITR 到 T1)
        │                    └─ timeline 3(PITR 到 T2)
        │
     timeline 1 继续
```

```sql
-- 当前时间线(通过 WAL 文件名可看出)
SELECT pg_walfile_name(pg_current_wal_lsn());   -- 前 8 位是时间线
SELECT timeline_id FROM pg_control_checkpoint();
```

### 清理关闭与崩溃关闭

| 关闭方式 | 行为 | 恢复成本 |
|----------|------|----------|
| `pg_ctl stop -m smart` | 等所有连接退出后关闭,做检查点 | 无恢复 |
| `pg_ctl stop -m fast`(默认) | 回滚进行中的事务,做检查点 | 无恢复 |
| `pg_ctl stop -m immediate` | 立刻退出,**不做检查点** | 下次启动需恢复 |
| 断电 / `kill -9` | 崩溃 | 从 redo point 恢复 |

> **`shutdown` 标记**:正常关闭会在控制文件写入标记。若启动时看不到该标记,说明上次是崩溃退出,**必然做恢复**。

## ③ 源码路径

| 内容 | 位置 |
|------|------|
| 检查点执行 | `src/backend/access/transam/xlog.c`(`CreateCheckPoint`) |
| 恢复主流程 | `src/backend/access/transam/xlogrecovery.c` |
| 控制文件 | `src/backend/access/transam/xlog.c`(`WriteControlFile`, `ReadControlFile`) |
| checkpointer 进程 | `src/backend/postmaster/checkpointer.c` |
| 页刷盘判断 | `src/backend/storage/buffer/bufmgr.c`(`FlushBuffer`, `BufferSync`) |
| WAL 摘要(PG17+) | `src/backend/access/transam/xlogrecovery.c`, `xlogprefetcher.c` |

## ④ 实验验证

### 实验 1:读控制文件

```bash
docker exec pg18 pg_controldata $PGDATA
# 或
docker exec -e PGDATA=/var/lib/postgresql/data pg18 pg_controldata
```

关注:

```
Database cluster state:               in production
Latest checkpoint location:           0/1A2B3C48
Latest checkpoint's REDO location:    0/1A2B3C00   ← 恢复起点
Latest checkpoint's TimeLineID:       1
```

### 实验 2:手工 CHECKPOINT 并观察

```sql
-- 看当前 WAL 位置
SELECT pg_current_wal_lsn() AS before;

CHECKPOINT;

SELECT pg_current_wal_lsn() AS after;
SELECT pg_control_checkpoint();
-- 输出:checkpoint_lsn, redo_lsn, timeline_id, prev_timeline_id,
--       full_page_writes, next_xid, ...
```

**`pg_control_checkpoint()`** 是查看检查点信息的 SQL 接口(无需文件访问)。

```sql
-- 其他控制文件信息
SELECT * FROM pg_control_system();
SELECT * FROM pg_control_init();
SELECT * FROM pg_control_recovery();   -- 恢复目标相关
```

### 实验 3:观察检查点对 WAL 与脏页的影响

```sql
-- 制造大量脏页
DROP TABLE IF EXISTS ckpt_demo;
CREATE TABLE ckpt_demo AS SELECT g AS id, repeat('x', 100) AS pad FROM generate_series(1, 1000000) g;
UPDATE ckpt_demo SET pad = pad || 'y';

-- 看脏页数(近似)
SELECT count(*) FILTER (WHERE isdirty) AS dirty_buffers
FROM pg_buffercache;

-- 强制检查点
CHECKPOINT;

SELECT count(*) FILTER (WHERE isdirty) AS dirty_buffers_after
FROM pg_buffercache;
-- 应显著减少
```

### 实验 4:模拟崩溃并观察恢复日志

```bash
# 1) 记录当前 LSN
docker exec pg18 psql -U postgres -c "SELECT pg_current_wal_lsn();"

# 2) 写入一些数据
docker exec pg18 psql -U postgres -c "CREATE TABLE crash_test (id int); INSERT INTO crash_test SELECT generate_series(1,1000);"

# 3) 强制崩溃(simulate crash)
docker exec pg18 bash -c "pg_ctl stop -m immediate -D \$PGDATA" || docker kill -9 pg18

# 4) 重启并查看日志
docker start pg18
docker logs pg18 2>&1 | tail -30
# 期望看到:
#   database system was interrupted; last known up at ...
#   redo starts at 0/...
#   consistent recovery state reached
#   redo done at 0/...
```

### 实验 5:验证 `pd_lsn` 与幂等重放

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- 页头 pd_lsn 记录最后一次修改它的 WAL 位置
SELECT lsn FROM page_header(get_raw_page('ckpt_demo', 0));

-- 对比当前 WAL 位置
SELECT pg_current_wal_lsn();
-- pd_lsn 应 <= 当前 WAL 位置
```

### 实验 6:验证检查点间隔警告

```sql
-- 缩短 max_wal_size 并持续写入,观察日志警告
ALTER SYSTEM SET max_wal_size = '64MB';
SELECT pg_reload_conf();

-- 持续写入并观察
DO $$
BEGIN
    FOR i IN 1..50 LOOP
        INSERT INTO ckpt_demo SELECT g, repeat('z',200) FROM generate_series(1,20000) g;
    END LOOP;
END $$;
```

```bash
docker logs pg18 2>&1 | grep -i "checkpoint" | tail -20
# 期望看到:checkpoints are occurring too frequently (Ns apart)
#           Consider increasing the configuration parameter "max_wal_size".
```

### 实验 7:测量恢复时间

```bash
# 制造大量未检查点的 WAL,然后崩溃
docker exec pg18 psql -U postgres -c "ALTER SYSTEM SET max_wal_size='4GB'; SELECT pg_reload_conf();"
docker exec pg18 psql -U postgres -c "INSERT INTO ckpt_demo SELECT g, repeat('w',200) FROM generate_series(1,5000000) g;"
# 立即崩溃(不等检查点)
docker kill -9 pg18
# 重启并计时
time docker start pg18
# 观察日志中的 redo 起止时间
```

## ⑤ 生产启示

| 结论 | 行动 |
|------|------|
| 恢复时间由检查点间隔决定 | **`max_wal_size` 越大,恢复越久**;需在"写入平滑"与"恢复速度"间权衡 |
| 检查点过于频繁是告警 | 日志出现 "too frequently" 就调大 `max_wal_size` 或 `checkpoint_timeout` |
| `completion_target` 别调小 | 保持 0.9 摊平 I/O;调小会造成周期性抖动 |
| 崩溃恢复期间不可用 | 大实例规划 RTO 时必须计入恢复时间;PG17+ 的 WAL 摘要能显著改善 |
| `immediate` 关闭会产生恢复 | 维护操作优先用 `fast` |
| 时间线管理是 PITR 的必修课 | PITR 后会产生新时间线,备库需相应处理 |
| 控制文件损坏是灾难 | 必须备份 `pg_control`(随基础备份一起) |

```sql
-- 检查点健康度监控
SELECT checkpoint_lsn, redo_lsn, timeline_id FROM pg_control_checkpoint();

-- 检查点频率(从日志或统计)
SELECT num_timed, num_requested, stats_reset
FROM pg_stat_checkpointer;      -- PG17+;PG16 及以前为 pg_stat_bgwriter 的部分列

-- pg_stat_bgwriter(旧版本)
SELECT checkpoints_timed, checkpoints_req, buffers_checkpoint,
       buffers_clean, maxwritten_clean, stats_reset
FROM pg_stat_bgwriter;
-- 若 checkpoints_req 远大于 checkpoints_timed → max_wal_size 太小
```

> **判断标准**:`checkpoints_req`(需求触发)应显著少于 `checkpoints_timed`(定时触发)。若相反,说明 `max_wal_size` 偏小,写负载把检查点"逼"出来了。

## 相关页面

- [[postgresql/03-进阶阶段-深度剖析/WAL与崩溃恢复/原理剖析-WAL与LSN机制]]
- [[postgresql/03-进阶阶段-深度剖析/复制与高可用内核/原理剖析-流复制与同步提交]]
- [[postgresql/02-中段阶段-专题精讲/备份恢复与高可用/专题概览]]
