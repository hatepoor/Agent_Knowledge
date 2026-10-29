---
tags: [mysql, 进阶, 日志, WAL, redo, binlog, checkpoint, GTID, 内核]
type: guide
created: 2026-10-13
updated: 2026-10-13
---

# 原理剖析 · redo 日志、WAL 与 binlog

> ⚠️ **本文是面试高频内容。** 「WAL 是什么」「为什么先写日志后写数据」「redo 和 binlog 有什么区别」都是必问。
> ⚠️ ⚠️ **⭐ 最后一节「为什么 binlog 要独立于 redo」是最高频面试题。**

## 学习目标

读完本篇，你应当能够：

1. ⭐ **说清 WAL（预写日志）的原理和必要性**
2. 解释 redo log 的结构和 `innodb_flush_log_at_trx_commit` 三个值
3. ⭐ **说清 checkpoint 是什么**以及它和 Buffer Pool 的关系
4. ⭐ **说清 redo log 和 binlog 的五个区别**（面试重点）
5. 理解 binlog 三种格式的写入差异
6. 理解 GTID 的实现机制

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[mysql/03-进阶阶段-深度剖析/事务与锁内核/原理剖析-undo日志与MVCC]] | 事务提交后的数据流转 |
| [[mysql/02-中段阶段-专题精讲/主从复制与高可用/复制原理与拓扑]] | binlog 的复制用途 |

---

## 一、⭐ WAL：预写日志

### 问题

⚠️ **如果直接写数据页：**

```
UPDATE t SET v = 2 WHERE id = 1;
    ↓
① 直接修改数据页（ibd 文件）
② ⚠️ 提交
    ↓
⛔ 崩溃了，磁盘上「写了一半」
```

⚠️ **⛔ 问题：⚠️ 写入过程被中断，磁盘上的数据页可能是「部分修改」状态——⚠️ 数据损坏。**

### WAL 的思路

⭐ **WAL = Write-Ahead Logging，⭐ 「写日志先行」。**

```
UPDATE t SET v = 2 WHERE id = 1;
    ↓
① ⭐ 先写 redo log（⚠️ 顺序写，快）
② 修改数据页（Buffer Pool 里，内存操作）
③ 提交
    ↓
⛔ 崩溃了？
    ├── 崩溃在 ① 之前 → 数据页没动，redo 没写 → ⛔ 未提交，符合预期
    ├── 崩溃在 ① 和 ② 之间 → ⭐ 有 redo 但数据页没改 → 恢复时重放 redo → ✅
    └── 崩溃在 ② 之后 → ⭐ 有 redo 且数据页已改 → 恢复时检测到（页 LSN == redo LSN）→ ✅ 跳过
```

⭐ **⭐ 核心思想：只要日志写成功了，即使数据页没来得及落盘，也能通过重放日志恢复——⚠️ 「日志」是持久化的保证，「数据页」只是最终结果。**

### 为什么这样更快

| 操作 | 特点 |
|------|------|
| ⭐ **redo log 顺序写** | ⭐ 顺序 IO 接近内存速度 |
| **数据页随机写** | ⛔ 随机 IO 是机械盘的性能杀手（SSD 上差距缩小） |

⚠️ **⚠️ 即使数据页最终要落盘，⚠️ 先写日志让它「可以晚点写」这个收益依然存在。**

📌 对照 PostgreSQL：⭐ **PG 的 WAL 也是这个思路**（XLog），⚠️ **PG 更进一步——binlog（逻辑日志）可以直接从 WAL（物理日志）生成**，⚠️ 所以 PG 只有一个日志。⚠️ **MySQL 为什么不能这么做，见第六节。**

---

## 二、redo log 的结构

### ⭐ 三个 LSN

| LSN | 全称 | 含义 |
|-----|------|------|
| ⭐ **Page LSN** | Log Sequence Number | ⚠️ **数据页头里记录的 LSN**——「这个页最后一次被修改时的日志位置」 |
| ⭐ **Redo LSN** | | ⚠️ **redo log 当前写到的位置** |
| ⭐ **Checkpoint LSN** | | ⚠️ **checkpoint 记录的位置**——「之前的日志都可以丢」 |

### ⭐ flush 判定的逻辑（重要）

```
某个数据页需要落盘时：
    ↓
比较 Page LSN 和 Redo LSN：
    ├── Page LSN < Redo LSN  → ⚠️ 落盘（这个页的改动还没在日志中反映完）
    └── Page LSN >= Redo LSN → ✅ 跳过（改动已经被 redo 覆盖，落盘也无所谓）
```

⭐ **📌 记忆锚点：redo 写在「数据页」前面，⚠️ 所以 redo 是「包含」数据页的。**

⚠️ **⚠️ 「页被刷脏了但 redo 还没刷盘」怎么办？** ⚠️ 这就是 doublewrite 解决的问题，见 [[mysql/03-进阶阶段-深度剖析/日志与复制内核/原理剖析-redo日志与WAL]]。

### `innodb_flush_log_at_trx_commit`

⚠️ **这是持久性的核心开关，面试必问。**

| 值 | 行为 | 崩溃（MySQL 进程死） | 崩溃（⚠️ 操作系统/机器死） | 性能 |
|---|------|-------------------|----------------------|------|
| ⭐ **`1`** | 每次提交都写 redo 到 OS 缓存 + fsync | ⛔ **不丢** | ⛔ **不丢** | ⚠️ 慢（每次提交一次 fsync） |
| **`2`** | 每次提交写 redo 到 OS 缓存，⚠️ 每秒由 MySQL fsync | ⛔ **不丢** | ⚠️ ⚠️ **丢 1 秒** | 中 |
| **`0`** | ⚠️ 每秒由 MySQL 写并 fsync | ⚠️ ⚠️ **丢 1 秒** | ⚠️⚠️ **丢 1 秒** | ⭐ 快 |

⚠️ **⚠️「操作系统崩溃丢 1 秒」的原因**：⚠️ **值 2 时，MySQL 每次提交只「写到 OS 缓冲区」不 fsync，⚠️ 数据在 MySQL 自己的内存里；MySQL 进程死了但 OS 还活着时，它每秒 fsync 一次能保住；⚠️ 但 OS 崩溃时，缓冲区里的数据还没落盘。**

### `sync_binlog`（binlog 的对应参数）

| 值 | 行为 |
|---|------|
| ⭐ **`1`** | ⭐ **每次提交都 fsync binlog 到磁盘**（⚠️ 最安全） |
| `N > 1` | 每 N 次提交 fsync 一次 |
| **`0`** | ⚠️ **交给 OS 决定何时刷**（⚠️ 主库宕机可能丢 binlog） |

⚠️ **⚠️ `sync_binlog=0` 特别危险**：⚠️ 如果主库宕机，⚠️ **binlog 丢了 → 从库永远补不上这段数据 → 永久性数据丢失**。

⭐ **📌 生产组合：`innodb_flush_log_at_trx_commit=1` + `sync_binlog=1`。⚠️ 代价是每次提交两次 fsync，性能下降约 30%。**

---

## 三、⭐ checkpoint

### 是什么

⚠️ **checkpoint 是「日志刷盘的基准点」。** ⭐ **它的含义是：⚠️ 这个 LSN 之前的所有 redo 都已经被应用到 Buffer Pool 并可以安全丢弃了。**

### Buffer Pool 与 redo 的循环

```
时间轴：
  ① UPDATE → 写 redo（LSN=100）→ 改 Buffer Pool 里的页
  ② checkpoint 记录 LSN=100 → 改 Buffer Pool 里的页
  ③ UPDATE → 写 redo（LSN=110）→ 改 Buffer Pool 里的页
  ...
  ⚠️ 当 Buffer Pool 里的脏页太多时（内存不足），
     需要刷一些页到磁盘才能继续 UPDATE
  ⛔ 刷盘时只能刷 LSN ≤ checkpoint 的页
```

⚠️ **⚠️ 这里有个关键约束**：⭐ **⚠️ 只能刷「Page LSN ≤ Checkpoint LSN」的页。** 因为：

```
如果刷了一个 Page LSN > Checkpoint LSN 的页：
    页已落盘（Page LSN=110）
    ⚠️ 但 checkpoint 还是 100
    ⛔ 崩溃后从 checkpoint=100 开始恢复 redo
    ⛔ 会重放 LSN=100~110 的 redo → ⚠️ ⚠️ 这些改动「已经落盘了」，重放会「多应用一次」！
```

⭐ **⚠️ 但 redo log 是幂等的**（写「把页的第 X 字节设为 Y」，不是「加 1」），⚠️ 所以理论上重放是安全的。⚠️ **⚠️ 但 InnoDB 还是用 checkpoint 避免了不必要的重放——⭐ 这也是 redo log 大小的设计依据。**

### ⚠️ checkpoint 推进受阻的情况

⭐ **checkpoint 只有在「对应的脏页都刷到磁盘了」时才能推进。**

⚠️ **⚠️ 如果有一个长事务一直持有老版本（undo 版本链很长），⚠️ purge 卡住，⚠️ checkpoint 就推不动 → redo log 写满了 → ⚠️ ⛔ 数据库「卡死」（不能写任何东西）。**

⚠️ **⛔ 表现**：应用报错 `ERROR 1114` 或者写操作 hang。

⚠️ **解法**：⚠️ **解决长事务**（见 [[mysql/02-中段阶段-专题精讲/事务与并发控制/长事务与undo膨胀]]）。

📌 详见 [[mysql/03-进阶阶段-深度剖析/日志与复制内核/原理剖析-redo日志与WAL]]。

---

## 四、binlog：为什么需要

⚠️ **redo log 有三个「致命限制」：**

| 限制 | 说明 |
|------|------|
| ⭐ **① 是 InnoDB 私有的** | MyISAM 没有 redo；换个引擎要重新实现复制 |
| ⭐ **② 是物理日志** | 「把 3 号页的第 5 个字节设为 0x11」——⚠️ 跨版本/跨平台不可移植 |
| ⭐ **③ 是循环写的** | ⚠️ 旧日志被覆盖，**不能用来做「按时间点恢复」** |

⚠️ **所以需要 binlog：**

| 需求 | binlog 满足 |
|------|------------|
| ⭐ **主从复制** | Server 层，与引擎无关 |
| ⭐ **按时间点恢复（PITR）** | 逻辑日志，有时间戳，且长期保留 |
| ⭐ **CDC（数据订阅）** | Canal/Debezium 读 binlog |
| 跨版本迁移 | 逻辑格式，更通用 |

---

## 五、⭐ redo vs binlog 五个区别（面试重点）

| # | 维度 | redo log（InnoDB） | binlog（Server 层） |
|---|------|------------------|-------------------|
| ⭐ **1** | **日志类型** | **物理日志**（页级字节修改） | **逻辑日志**（SQL 或行变化） |
| ⭐ **2** | **作用** | 崩溃恢复 | 主从复制 + PITR + CDC |
| ⭐ **3** | **写入方式** | **循环写**（写满覆盖） | **追加写**（⭕ 永不覆盖，按文件） |
| ⭐ **4** | **写时机** | 事务提交时 | ⚠️ **`COMMIT` 时**（配合两阶段提交） |
| ⭐ **5** | **所属层** | **InnoDB 引擎层** | **MySQL Server 层** |

### 补充区别

| 维度 | redo log | binlog |
|------|---------|--------|
| 空间 | ⭐ 环形缓冲，大小固定（`innodb_redo_log_capacity`） | ⚠️ 追加文件，⚠️ 可以很大 |
| 内容 | 页的字节级修改 | `ROW`：每行的 before/after |
| 刷盘参数 | `innodb_flush_log_at_trx_commit` | `sync_binlog` |
| 能否被读 | ⭐ **不能被外部工具读** | ⭐ **`mysqlbinlog` 能读** |
| 归档 | ❌ 自动覆盖 | ⭐ `expire_logs_days` / `binlog_expire_logs_seconds` |

### ⭐⭐ 两阶段提交（面试高频）

⚠️ **这是一个必须理解的设计，否则理解不了「为什么两个日志要同步」。**

```
场景：如果 redo 写了但 binlog 没写（或反之），会怎样？
```

**只写 redo，不写 binlog**：⚠️ 主库有这行数据，但**从库没有**，⚠️ 从主库恢复出来的实例也没有——⛔ **不一致**。
**只写 binlog，不写 redo**：⚠️ 从库有这行数据，⚠️ **但主库崩溃后恢复不出来——⛔ 主从数据「凭空多出来」**。

⭐ **解决方案：两阶段提交**

```
阶段 1：redo log 标记「准备提交」（prepare）
        ↓
阶段 2：写 binlog
        ↓
阶段 3：redo log 标记「提交完成」（commit）
```

⭐ **崩溃恢复时的判定**：

```
发现 redo 是「prepare」状态
    ↓
⚠️ 检查 binlog 是否完整（有相同事务 ID 的记录？）
    ├── 是 → ⛔ redo 标记为 commit（提交）
    └── 否 → ⛔ redo 标记为 rollback（回滚）
```

⚠️ **⚠️ 这是「redo 和 binlog 必须同步写」的机制保证。**

📌 详解见 [[mysql/03-进阶阶段-深度剖析/日志与复制内核/原理剖析-redo日志与WAL]]。

---

## 六、⭐⭐⭐ 面试题：为什么 binlog 要独立于 redo？

⚠️ **⚠️ 这是 MySQL 面试最高频的一道原理题。标准答案有三个层次：**

### 层次 1：历史原因

⚠️ **⚠️ 最开始 MySQL 只有 binlog（那时叫 query log），⚠️ 是「MyISAM 的日志」，⚠️ 只能做主从复制，⚠️ 不能做崩溃恢复。**

⚠️ **⚠️ 后来 Oracle 收购 MySQL，引入了 InnoDB（已有的第三方引擎），⚠️ InnoDB 需要「崩溃恢复」能力，⚠️ 就自己加了一套 redo log。**

⚠️ **⭐ 结果就是：⚠️ 「复制用 binlog（Server 层）+ 恢复用 redo（引擎层）」的两套日志并存。** ⚠️ **不是设计出来的，是历史演进的产物。**

⚠️ **⚠️ 而且 binlog 在 InnoDB 之前就有了，⚠️ 所以它不能改（⚠️ 改了就和所有已有的从库不兼容）。**

### 层次 2：架构原因（⭐ 核心）

⚠️ **三个层面的必然性：**

| 层面 | 原因 |
|------|------|
| ⭐ **① 引擎无关** | ⚠️ binlog 是 **Server 层**的，⚠️ 所有引擎（InnoDB、MyISAM、MEMORY…）共用。⚠️ redo 是 **InnoDB 私有**的 |
| ⭐ **② 复制需要可读** | ⚠️ **redo 是物理日志**（「3 号页第 5 字节设为 0x11」），⚠️ 从库版本不同就可能无法应用。⚠️ binlog 是**逻辑日志**，跨版本友好 |
| ⭐ **③ 恢复需要循环日志** | ⚠️ redo **循环写**（空间固定，写满就覆盖旧的）——⚠️ 这是为了「崩溃恢复只关心最近的变化」。⚠️ binlog **追加写**（长期保留），⚠️ 这是为了「PITR 需要历史数据」 |

### 层次 3：PostgreSQL 的对照（⭐ 加分项）

⭐ **PG 只有一个日志（WAL），⚠️ 所以 binlog 逻辑日志是从 WAL 物理日志「解码」出来的。**

```
PG：物理 WAL → 解码 → 逻辑流 → 流复制
MySQL：redo（物理，循环） + binlog（逻辑，追加）  ⚠️ 两份独立写
```

⚠️ **⚠️ 代价**：MySQL 的两阶段提交。⚠️ **⚠️ PG 因为只有一个日志，不存在「两个日志要同步」的问题。**

⭐ **📌 一句话总结答案**：
> **「⚠️ 因为 MySQL 的 binlog 比 InnoDB 更早存在，且 binlog 要服务所有引擎（引擎无关）、要跨版本复制（逻辑格式）、要长期保留做 PITR（追加写）；而 redo 只服务 InnoDB 自己的崩溃恢复，物理格式、循环写即可。PostgreSQL 只有一个 WAL，所以没这个问题。」**

---

## 七、binlog 三种格式的写入

| 格式 | 写入内容 | ⚠️ 问题 |
|------|---------|--------|
| `STATEMENT` | SQL 语句原文 | ⛔ `NOW()`、`UUID()`、`RAND()`、`LIMIT` 无序 → **主从不一致** |
| ⭐ `ROW` | ⭐ **每行的 before image + after image** | ✅ 必定一致 |
| `MIXED` | 自动选择 | ⚠️ 行为不够可预测 |

⚠️ **⚠️ `ROW` 的三个参数**：

| 参数 | 说明 |
|------|------|
| ⭐ `binlog_row_image` | `FULL`（默认）/ `MINIMAL` / `NONE`。⚠️ **`MINIMAL` 只记变化的列，⚠️ CDC 工具会失败** |
| `binlog_row_metadata` | `MINIMAL`（8.0 默认）/ `FULL`。⚠️ **`FULL` 含列名，⚠️ CDC 场景需要** |

📌 详见 [[mysql/02-中段阶段-专题精讲/主从复制与高可用/复制原理与拓扑]]。

---

## 八、GTID 的实现

### 结构

```
GTID = UUID:序列号
例：3e11fa47-71ca-11e1-9e33-c80aa9429562:1-23
    └─── 服务器 UUID ───┘ └─ 事务编号范围 ─┘
```

⭐ **每个事务在整个集群里有唯一编号。**

### 为什么需要

⚠️ **传统 binlog 定位用「文件名 + 偏移量」：**

```
主库 binlog.000005 位置 1234
    ↓
主库 failover → 新主库的 binlog.000002 位置 5678
    ↓
⚠️ 从库配置要全部改，且容易出错
```

⭐ **GTID 的解法**：⭐ **从库只需要知道主库的 UUID**，⚠️ **自动从「已执行的事务」之后继续。**

### 执行集合

```sql
SELECT * FROM information_schema.GTID_EXECUTED\G
-- +-------------------------------------------+
-- | 3e11fa47-...:1-23                         |
-- +-------------------------------------------+

SELECT @@GLOBAL.gtid_executed;   -- 同上
SELECT @@GLOBAL.gtid_purged;     -- 已删除 binlog 对应的事务集合
```

⭐ **`gtid_purged` 是新概念**：⚠️ 当你删除 binlog 文件后，⚠️ **MySQL 要把「这些事务已执行」记下来**，⚠️ 否则从库来请求这些 binlog 时会出错。

### 三个前提

⚠️ **必须同时满足**：

```ini
gtid_mode = ON
enforce_gtid_consistency = ON
binlog_format = ROW               -- ⭐ 必须
```

⚠️ **`gtid_mode` 是不可逆的**（⚠️ 8.0 里是只读参数，只能 ON）。

📌 完整用法见 [[mysql/02-中段阶段-专题精讲/主从复制与高可用/复制原理与拓扑]]。

---

## 九、⭐ 实验清单

### 实验 1：观察 redo log 位置

```sql
-- ⭐ 看 checkpoint LSN 和 redo LSN
SHOW ENGINE INNODB STATUS\G
-- 看 "LOG" 段落：
--   Log sequence number 12345678
--   Checkpoint LSN      12345000
--   ⚠️ 两者的差就是「还没落盘的量」
```

### 实验 2：验证 `innodb_flush_log_at_trx_commit` 的性能差异

```sql
-- 造 1 万行数据
CREATE TABLE perf (id INT PRIMARY KEY AUTO_INCREMENT, v INT);
-- 用 sysbench 或循环插入

-- 测两次
SET GLOBAL innodb_flush_log_at_trx_commit = 1;    -- 安全
-- 跑插入，记录耗时

SET GLOBAL innodb_flush_log_at_trx_commit = 0;    -- ⚠️ 不安全
-- 跑插入，记录耗时
-- ⚠️ ⚠️ 对比差异，通常 30%~50%
```

⚠️ **⚠️ 这个实验只在测试环境做**（⚠️ 会真的丢失数据）。

### 实验 3：⭐ 观察崩溃恢复（⚠️ 只在 Docker 里）

```bash
# 在 Docker 容器里
docker kill --signal=KILL mysql-lab      # ⭐ 模拟进程被杀（模拟崩溃）
docker start mysql-lab

docker logs mysql-lab                    # ⭐ 看恢复过程
# ⚠️ 会看到 "Starting recovery for InnoDB" 等信息
```

⚠️ **⚠️ 实验 3 必须在 Docker 里做**，⚠️ **⚠️ 绝对不要 kill 你的开发实例。**

### 实验 4：对比 redo 和 binlog

```sql
-- 观察 redo 循环写
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';
SHOW ENGINE INNODB STATUS\G              -- 看 Log sequence number 持续增长

-- 观察 binlog 追加写
SHOW BINARY LOGS;                          -- ⚠️ 文件一直增加
SHOW VARIABLES LIKE 'log_bin_basename';
```

⚠️ **⭐ 关键观察**：⚠️ **redo LSN 会超过容量后「绕回」（mod capacity），binlog 文件号只增不减。**

### 实验 5：⭐ PG 对照（理解「一个日志 vs 两个日志」）

```sql
-- PG 上
SHOW wal;
-- ⚠️ 只有一个 WAL，同时用于恢复和流复制
SELECT * FROM pg_stat_replication;   -- 用同一个日志流
```

---

## 十、陷阱

1. **❌ 以为 redo log 能用于主从复制。** ⚠️ 它是引擎私有、物理格式、循环写。
2. **❌ 以为 binlog 能用于崩溃恢复。** ⚠️ 它是逻辑日志，恢复慢且不完整。
3. **❌ 生产把 `innodb_flush_log_at_trx_commit` 设成 0/2。** ⚠️ 丢 1 秒数据。
4. **❌ 生产把 `sync_binlog` 设成 0。** ⚠️ ⚠️ **主库宕机可能丢 binlog，⚠️ 从库永远补不上**。
5. **❌ 以为 checkpoint 能随便推进。** ⚠️ 只能刷 Page LSN ≤ Checkpoint LSN 的页。
6. **❌ 说不清「为什么 binlog 独立于 redo」。** ⚠️ 这是面试高频题，见第六节。
7. **❌ 用 `binlog_row_image = MINIMAL` 但要跑 CDC。** ⚠️ 工具会失败。
8. **❌ 开 GTID 不用 `ROW` 格式。** ⚠️ 启动会报错。
9. **❌ 在重要实例上做崩溃实验。** ⚠️ 只在 Docker 里。

---

## 小结

- ⭐ **WAL（预写日志）**：⭐ **「日志先行」——先写 redo 再改数据页，⚠️ 顺序写比随机写快，⭐ 且崩溃后能靠重放日志恢复**
- **三个 LSN**：Page LSN（页头）、Redo LSN（当前写位置）、Checkpoint LSN（安全线）
- ⚠️ **刷盘判定**：Page LSN < Redo LSN 才需要刷盘
- ⚠️ **`innodb_flush_log_at_trx_commit`**：`1`（生产，安全）/ `2`（MySQL 崩溃不丢，⚠️ **OS 崩溃丢 1 秒**）/ `0`（都可能丢）
- ⚠️ **`sync_binlog=1`** 同样重要——⚠️ **丢了 binlog 从库永远补不上**
- ⭐ **checkpoint**：⚠️ 「之前的 redo 可以丢弃」的基准点；⚠️ **被长事务卡住 → redo 写满 → 数据库卡死**
- ⭐⭐ **redo vs binlog 五个区别**：物理/逻辑、恢复/复制、循环/追加、InnoDB/Server、PageLSN/追加
- ⭐⭐ **两阶段提交**：redo prepare → 写 binlog → redo commit；⚠️ 崩溃时靠「binlog 是否完整」判定
- ⭐⭐⭐ **面试题「为什么 binlog 独立于 redo」的三层答案**：⚠️ **① 历史（binlog 更早，redo 是后加的）② 架构（引擎无关 + 逻辑格式跨版本 + 追加写支持 PITR）③ 对照 PG（PG 只有一个 WAL 所以没这问题）**
- **GTID**：`UUID:序号`；⚠️ 需要 `ROW` 格式，⚠️ `gtid_mode` 不可逆

## 练习

1. 什么是 WAL？为什么先写日志能提速又能保证安全？
2. Page LSN、Redo LSN、Checkpoint LSN 分别是什么？刷盘时怎么用它们判定？
3. `innodb_flush_log_at_trx_commit` 的三个值分别意味着什么？为什么 0 和 2 有区别？
4. 为什么 `sync_binlog=0` 比 `innodb_flush_log_at_trx_commit=0` 更危险？
5. checkpoint 是什么？为什么它被长事务卡住会导致数据库「卡死」？
6. ⭐⭐ 完整说出 redo log 和 binlog 的五个区别。
7. ⭐⭐⭐ 「为什么 MySQL 需要 binlog 而不能直接用 redo 做复制？」——说出三个层次的理由。
8. 为什么需要「两阶段提交」？只写 redo 或只写 binlog 分别会导致什么不一致？
9. PostgreSQL 为什么只有一个日志？这给了你什么启发？
10. GTID 解决了什么问题？为什么它依赖 `ROW` 格式？

## 导航

[[mysql/03-进阶阶段-深度剖析/00-剖析方法论|剖析方法论]] · 上一篇：[[mysql/03-进阶阶段-深度剖析/事务与锁内核/原理剖析-锁体系与死锁]] · 下一篇：[[mysql/03-进阶阶段-深度剖析/优化器内核/原理剖析-优化器四阶段与成本模型]] · 相关：[[mysql/02-中段阶段-专题精讲/主从复制与高可用/复制原理与拓扑|复制原理与拓扑]]
