---
tags: [mysql, 中段, 长事务, undo, purge, 性能]
type: guide
created: 2026-10-13
updated: 2026-10-13
---

# 长事务与 undo 膨胀

> **什么时候该读**：**「数据库越用越慢」的第一个排查方向。** 磁盘持续增长、purge 卡住、查询变慢——十有八九是长事务。

## 学习目标

读完本篇，你应当能够：

1. 说清「长事务」的两种形态和它的四大危害
2. 解释 undo 表空间膨胀和 `history list length` 的关系
3. 掌握定位长事务的三个查询
4. 给出治理措施（预防 + 排查 + 应急）
5. 理解为什么长事务会导致主从延迟
6. 会调整 undo 相关参数

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[mysql/01-入门阶段-基础教程/第04章-事务与权限/4.1-事务与ACID]] | 事务、undo log、隔离级别 |
| [[mysql/02-中段阶段-专题精讲/InnoDB存储引擎/专题概览]] | purge 线程与 Buffer Pool |

---

## 一、长事务的两种形态

### 形态 1：「长时间不提交的事务」⚠️ 最常见

```python
# Python 侧最常见的来源
conn = pymysql.connect(...)          # ⚠️ autocommit=False
try:
    cur = conn.cursor()
    cur.execute("SELECT ...")         # 事务开了，但没 commit
    # ⚠️ 代码出异常 / 忘了 / 走了某个分支没提交
except:
    pass                             # ⚠️ 没有 rollback，也没有 commit
# 连接被借回连接池，⚠️ 事务还挂着
```

⚠️ **⚠️ 这是生产数据库长事务的头号来源。** 连接池把带未提交事务的连接借回池里，下一个借用者可能遇到「上一个业务的数据还没提交」或「连接已经被标记为事务中」。

**其他来源**：

| 来源 | 说明 |
|------|------|
| 交互式客户端忘了 | 用 `mysql` 命令行时忘了 `COMMIT` |
| 存储过程里没提交 | |
| 批处理脚本中途失败 | ⚠️ 异常被吞了但没 rollback |
| `START TRANSACTION` 后只做只读 | ⚠️ 快照会一直挂着 |
| ⚠️ **DDL 之前的隐式事务** | `START TRANSACTION; CREATE TABLE...;` → ⚠️ CREATE 隐式提交了，但后续的事务状态可能异常 |

### 形态 2：「执行耗时长的 SQL」

```sql
-- 一条跑 10 分钟的 UPDATE
UPDATE big_table SET status = 1 WHERE status = 0;
```

⚠️ **⚠️ 这类长事务不会「挂着」，但它执行期间会：**
- 持有大量行锁
- 产生大量 undo
- **在从库上阻塞整个复制**（⚠️ 单线程回放）

---

## 二、四大危害

### 危害 1：undo 表空间膨胀 ⚠️

### undo log 做什么

```sql
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
```

InnoDB 在改数据**之前**，先把「怎么撤销」写进 undo：

| 操作 | undo 记录 |
|------|----------|
| `INSERT` | ⭐ 记「这条是新增的，撤销=删除」 |
| `DELETE` | 记「被删的行原值，撤销=插回」 |
| `UPDATE` | 记「哪些列的旧值，撤销=改回」 |

### ⚠️ 关键机制：purge 线程

```
事务 A 开启时（创建 ReadView）
    ↓
之后每个事务的每次修改都产生 undo
    ↓
purge 线程（后台）尝试清理「不再被任何事务需要」的 undo
    ↓
⚠️ 但只要还有事务持有旧的 ReadView，purge 就不能清理它之后的 undo
```

⭐ **这就是「history list length」的含义**：undo 链上「最老的事务」到「现在」之间累积的 undo 记录数。

```sql
SHOW ENGINE INNODB STATUS\G
```

看这一行：

```
History list length 234567
```

⚠️ **⚠️ 这是最重要的监控指标**：

| 值 | 判断 |
|---|------|
| < 10,000 | ✅ 正常 |
| 10,000 ~ 100,000 | ⚠️ 关注 |
| > 100,000 | ❌ ❗ **有问题，需立即处理** |
| > 1,000,000 | ⚠️⚠️ **严重，purge 基本卡死** |

⚠️ **⚠️ 长事务会让这个值持续增长，且长事务提交后 purge 还要「追赶」。** 这期间：
- undo 表空间膨胀（占磁盘）
- ⭐ **其他事务的 MVCC 读要遍历很长的版本链** → 变慢
- **磁盘写满** → 数据库不可用

### 危害 2：阻塞 purge 线程 ⚠️⚠️

⭐ **这是长事务最隐蔽的危害。** 展开说：

```
时间轴：
10:00  事务 A 开启（ReadView 指向 10:00）
10:00~10:30  事务 B/C/D... 写了 5000 万行
             undo 链有 5000 万条
             ↓
             purge 想清理，但发现 A 的 ReadView 可能还要看 10:00 之后的数据
             ⛔ 无法清理
             ↓
10:30  事务 A 提交
       ⚠️ purge 现在才开始追赶清理 5000 万条
       ⚠️ 清理期间 undo 表空间一直是大的
       ⚠️ 而且清理本身要消耗 IO
```

⚠️ **结果**：即使长事务结束了，数据库还会「不舒服」很久。

⚠️ **⚠️ 更糟的情况**：如果 A 从不提交（连接泄漏、代码 bug），purge **永远不会恢复**，undo 无限增长。

### 危害 3：锁阻塞

```sql
-- 会话 1（长事务）
BEGIN;
UPDATE orders SET status='processing' WHERE id = 1;
-- ⚠️ 没提交，持有锁

-- 会话 2
UPDATE orders SET status='done' WHERE id = 1;
-- ⛔ 等待锁超时（默认 innodb_lock_wait_timeout = 50 秒）
```

⚠️ **长事务持有的锁越多，并发能力下降越明显。**

📌 排查见 [[mysql/03-进阶阶段-深度剖析/事务与锁内核/原理剖析-锁体系与死锁]]。

### 危害 4：主从延迟 ⚠️⚠️

⚠️ **这个经常被忽视。**

```
主库：执行一个 10 分钟的大事务
  ↓
binlog 里对应一个大事务（或很多）
  ↓
从库：⚠️ 单线程回放 binlog，遇到这个大事务时
      ⚠️ ⛔ 从库上其他事务都无法并行应用
      ↓
      ⛔ 从库落后 10 分钟
```

⚠️ **⚠️ MySQL 5.7+ 有并行复制（`slave_parallel_type` / `slave_parallel_workers`），⚠️ 但主线程回放大事务时依然不能并行。**

⚠️ **所以：从库上跑慢查询或大批量操作，会导致严重主从延迟。** ⚠️ 而且**这个延迟不能靠「加速回放」消除**——必须等这个事务完成。

📌 主从延迟排查见 [[mysql/02-中段阶段-专题精讲/主从复制与高可用/复制原理与拓扑]]。

---

## 三、定位长事务

### 查询 1：当前正在运行的事务

```sql
SELECT * FROM information_schema.INNODB_TRX\G
```

关键字段：

| 字段 | 含义 |
|------|------|
| `trx_id` | 事务 ID |
| `trx_state` | `RUNNING` / `LOCK WAIT` |
| `trx_started` | ⚠️ **开始时间（算时长的依据）** |
| `trx_duration` | ⚠️ **已持续秒数**（8.0 有这个字段） |
| `trx_mysql_thread_id` | ⭐ **连接 ID，用它去 PROCESSLIST 找人** |
| `trx_query` | ⚠️ 当前/最近的 SQL（⚠️ 只读事务时是 NULL） |
| `trx_rows_locked` | 锁住的行数 |
| `trx_rows_modified` | 修改的行数 |
| `trx_isolation_level` | 隔离级别 |

⚠️ **`trx_query` 是 NULL 不代表没问题**——⚠️ **空闲但开着事务的连接是最危险的**（`trx_query` 为 NULL，但事务还挂着）。

### 查询 2：找出超过 N 秒的事务 ⭐

```sql
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_sec,
       trx_mysql_thread_id AS conn_id, trx_query
FROM information_schema.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60
ORDER BY trx_started;
```

### 查询 3：⚠️ 空闲但开着事务的连接（头号问题）⭐

```sql
SELECT t.trx_id,
       TIMESTAMPDIFF(SECOND, t.trx_started, NOW()) AS duration_sec,
       p.id AS conn_id, p.user, p.host, p.command, p.time AS idle_sec, p.state
FROM information_schema.INNODB_TRX t
JOIN information_schema.PROCESSLIST p ON p.id = t.trx_mysql_thread_id
WHERE p.command = 'Sleep'         -- ⭐ 关键：连接空闲但事务还开着
ORDER BY t.trx_started;
```

⭐ **⚠️ 这是最该定期跑的监控 SQL。** `command = 'Sleep'` 且在 `INNODB_TRX` 里的连接，就是「空闲但开着事务」的泄漏。

⚠️ **⚠️ 这种情况最危险**——客户端可能已经「认为」事务结束了（比如超时），但服务端还在挂着，ReadView 一直不释放。

### 查询 4：history list length

```sql
SHOW ENGINE INNODB STATUS\G
-- 找 "History list length" 这一行
```

⚠️ **⚠️ 也可以从 `information_schema.INNODB_METRICS` 拿（更轻量）**：

```sql
SELECT count FROM information_schema.INNODB_METRICS
WHERE name = 'trx_rseg_history_len';
```

⭐ **`trx_rseg_history_len` 就是 history list length**，⚠️ **这是最适合做监控告警的指标**（每秒采样即可，不用跑 `SHOW ENGINE INNODB STATUS`）。

### 查询 5：undo 表空间大小

```sql
SELECT tablespace_name, file_size, file_size / 1024 / 1024 AS size_mb
FROM information_schema.FILES
WHERE file_type = 'UNDO LOG';
```

⚠️ **⚠️ 8.0 默认有独立的 undo 表空间**（`undo_001`、`undo_002`），⚠️ 可以在线扩容（但⚠️ 不能缩容）。

```sql
-- 8.0 加 undo 表空间
ALTER TABLESPACE innodb_undo_001 ADD DATAFILE 'undo_003' SIZE 128M;
```

📌 完整机制见 [[mysql/03-进阶阶段-深度剖析/事务与锁内核/原理剖析-undo日志与MVCC]]。

### 定位到人

```sql
-- 拿到 conn_id 后
SELECT id, user, host, db, command, time, state, info
FROM information_schema.PROCESSLIST
WHERE id = 12345;
```

⚠️ **`host` 字段能定位到哪台机器、哪个端口**，顺着应用日志能查到具体是哪个请求。

⚠️ **⚠️ 注意权限**：`SHOW PROCESSLIST` 默认只能看自己的连接，需要 `PROCESS` 权限才能看全部。

```sql
GRANT PROCESS ON *.* TO 'monitor'@'%';
```

---

## 四、治理措施

### 预防：从源头减少长事务

#### 措施 1：⚠️ 缩短事务时间 ⭐

```sql
-- ❌ 一个大事务处理 1 万条
BEGIN;
INSERT INTO target SELECT * FROM source WHERE id BETWEEN 1 AND 10000;
COMMIT;

-- ✅ 分批，每批一个事务
INSERT INTO target SELECT * FROM source WHERE id BETWEEN 1 AND 1000;
INSERT INTO target SELECT * FROM source WHERE id BETWEEN 1001 AND 2000;
...
```

⚠️ **⚠️ 分批的批大小**：⚠️ 通常 500~2000 行，⚠️ **视单行大小和 undo 压力调整**。

⭐ **📌 记忆锚点：事务不是越长越好。短事务对系统更友好。**

#### 措施 2：不要在事务里做「非数据库操作」

```python
# ❌ 危险：网络请求在事务里
conn.begin()
cur.execute("SELECT ...")
requests.post("http://external-service/...", timeout=10)  # ⚠️ 可能 hang 30 秒
cur.execute("UPDATE ...")
conn.commit()
```

⚠️ **外部调用耗时会直接拉长事务。** ⭐ **正确顺序：先做外部调用，再开事务做数据库操作。**

#### 措施 3：只读操作用自动提交

```python
# ✅ 只读查询不需要事务
conn = pymysql.connect(..., autocommit=True)
# 或
cur.execute("START TRANSACTION READ ONLY")
```

⚠️ **⚠️ `START TRANSACTION READ ONLY`**：⚠️ MySQL 5.6+ 支持「只读事务」，⚠️ 它不创建 ReadView（⚠️ 但仍会占用一个事务槽位）。

⚠️ **更简单：只读查询就用 autocommit，不开事务。**

#### 措施 4：⚠️ 应用侧设置超时

```python
# ✅ 客户端超时
conn = pymysql.connect(..., read_timeout=30, write_timeout=30)

# ✅ 或在语句级别
cur.execute("SET SESSION innodb_lock_wait_timeout = 5")
```

⚠️ **⚠️ 更根本的做法**：应用层加「事务时长监控」，超过阈值告警。

### 排查：定期巡检

```sql
-- 建议每分钟跑一次并告警
-- ① history list length
SELECT count AS history_len FROM information_schema.INNODB_METRICS
WHERE name = 'trx_rseg_history_len';

-- ② 超过 30 秒的事务数
SELECT COUNT(*) AS long_trx_count
FROM information_schema.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 30;
```

| 告警阈值 | 建议 |
|---------|------|
| `history_len > 100000` | ⚠️ 警告 |
| `history_len > 1000000` | ❌ 紧急 |
| `long_trx_count > 0`（超过 30 秒的） | ⚠️ 警告 |
| **存在 `command='Sleep'` 且在 INNODB_TRX 的连接** | ❌ **紧急** |

### 应急：处理正在发生的长事务

#### 手段 1：KILL 连接（⚠️ 会回滚）

```sql
-- 找到 conn_id
SELECT trx_mysql_thread_id FROM information_schema.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 3600;   -- 超过 1 小时

-- ⚠️ KILL（MySQL 8.0）
KILL CONNECTION 12345;      -- 只断连接，不中止语句
KILL 12345;                 -- 中止语句

-- ⚠️ MySQL 5.7 用 KILL QUERY（只中止查询，不断连接）
KILL QUERY 12345;
```

⚠️ **⚠️ 代价**：

| 代价 | 说明 |
|------|------|
| 未提交的事务被回滚 | ⚠️ 数据丢失（对应用来说是「操作失败」） |
| ⚠️ 回滚耗时可能很长 | ⚭� 大事务回滚要逐条反向操作 |
| 应用会收到异常 | 需要应用有重试机制 |

⚠️ **⚠️ 只有在确认「这个事务卡死了，且没人负责」时才 KILL。** ⚠️ **KILL 一个大事务可能引发比原问题更严重的后果**（回滚也要很久）。

#### 手段 2：等它自己结束

⚠️ **如果事务是正常业务（只是慢），让它跑完。** ⚠️ 杀掉反而更糟。

⭐ **判断标准**：

| 情况 | 处理 |
|------|------|
| `trx_query` 有 SQL，且时间在增长 | ⚠️ **正常业务在跑 → 等** |
| `command='Sleep'` + 在 INNODB_TRX | ❌ **连接泄漏 → KILL** |
| `trx_query` 长时间不变（但 `command` 不是 Sleep） | ⚠️ 可能卡在锁等待 → 查锁等待 |
| 时间以「小时」计 | ❌ 大概率是 bug → 排查应用 |

### 参数调整

#### `innodb_undo_tablespaces` 与 `innodb_undo_directory`

⚠️ **MySQL 8.0 默认独立 undo 表空间**：

```sql
SHOW VARIABLES LIKE 'innodb_undo%';
```

| 参数 | 默认 | 说明 |
|------|------|------|
| `innodb_undo_tablespaces` | ⚠️ **8.0 固定为 2** | ⚠️ 8.0 里不能改（⚠️ 5.7 可改） |
| `innodb_undo_directory` | `./` | 存放位置 |
| `innodb_max_undo_log_size` | 1GB | ⚠️ 每个 undo 文件的上限 |

⭐ **8.0 的改进：undo 表空间可以在线扩容**（⚠️ 5.7 不行）：

```sql
ALTER TABLESPACE innodb_undo_001 ADD DATAFILE 'undo_003' SIZE 256M;
```

⚠️ **⚠️ 但 undo 表空间过大本身也是问题**——⚠️ 说明有长事务。⚠️ **扩容只是应急，不是解法。**

#### `innodb_purge_threads` 与 `innodb_purge_batch_size`

```sql
SHOW VARIABLES LIKE 'innodb_purge%';
-- innodb_purge_threads = 4          # 默认 4
-- innodb_purge_batch_size = 300     # 每批清理多少条
```

⚠️ **⚠️ 调大可以加快 purge 追赶**，⚠️ 但会消耗更多 IO。

⚠️ **⚠️ 调整建议**：

| 场景 | 建议 |
|------|------|
| ⚠️ 平时 | **不要动**（默认值平衡了 CPU 和 IO） |
| ⚠️ 发现 purge 长期跟不上（有长事务刚结束） | ⚠️ 临时调大 `innodb_purge_batch_size`（如 1000） |

⚠️ **⚠️ 更根本的做法是解决长事务**，⚠️ 调参数只能缓解。

#### `max_execution_time`

```sql
-- 限制单条 SELECT 的最大执行时间（毫秒）
SET GLOBAL max_execution_time = 5000;      -- 5 秒

-- 只对特定会话生效（不误伤内部线程）
SET SESSION max_execution_time = 3000;
```

⚠️ **⚠️ 这个只管 SELECT，不管 UPDATE/DELETE。** ⚠️ MySQL 没有「限制 DML 执行时间」的参数。

⚠️ **⚠️ 注意**：⚠️ 内部线程（复制、健康检查）默认不受影响，⚠️ 但如果你设了全局值，⚠️ **⚠️ 应用侧可能因为超时被 KILL。**

---

## 五、⚠️ 常见误区

### 误区 1：「大查询不等于长事务」

⚠️ **要区分**：

| 情况 | 是否长事务 |
|------|-----------|
| 一条 SELECT 跑 5 分钟（无显式事务） | ⚠️ **不是**（autocommit 模式下没有长事务） |
| `BEGIN; SELECT(跑 5 分钟); COMMIT;` | ⚠️ **是**（ReadView 挂了 5 分钟） |
| 一条 UPDATE 跑 5 分钟 | ⚠️ **是**（执行期间就是长事务） |

⭐ **但两者都有害**：前者拖慢查询和 CPU，后者还额外影响 undo 和复制。

### 误区 2：「事务越长越好，减少提交次数」

❌ **完全错误。** ⚠️ 频繁提交的开销远小于长事务的代价（fsync + undo 膨胀 + purge 卡住 + 主从延迟）。

⭐ **正确的原则**：**事务「尽可能短」，但也不要「每个语句一个事务」**。一个业务操作（哪怕包含几条 SQL）放在一个事务里，这是合理的粒度。

### 误区 3：「批量操作必须在一个事务里保证原子性」

⚠️ **不一定。** ⚠️ 如果是「数据迁移」类的批处理，⚠️ **允许部分成功 + 记录失败项 + 重试**，通常比「一个大事务」更好。

📌 迁移方案见 [[mysql/02-中段阶段-专题精讲/大表与在线DDL/在线DDL方案对比]]。

---

## 六、陷阱

1. **❌ 不监控 `trx_rseg_history_len`。** ⚠️ 它是长事务的最灵敏指标。
2. **❌ 不监控「空闲但开着事务」的连接。** ⚠️ 这是头号泄漏源。
3. **❌ 在事务里调外部接口。** ⚠️ 网络 hang 会拉长事务。
4. **❌ 大批量操作不���批。** ⚠️ 一个大事务。
5. **❌ 看到大事务就 KILL。** ⚠️ 回滚耗时可能更长，要先判断是不是正常业务。
6. **❌ 只扩容 undo 不解决长事务。** ⚠️ 应急手段，不是解法。
7. **❌ 以为只读查询不会有长事务问题。** ⚠️ `START TRANSACTION` + SELECT 一样会挂 ReadView。
8. **❌ Python 侧忘 commit / 忘 rollback。**

---

## 小结

- 长事务两种形态：**① 长时间不提交**（⚠️ 头号来源是连接泄漏）、**② 耗时长的 SQL**
- ⚠️ **四大危害**：undo 膨胀、**阻塞 purge 线程**（⚠️ 最隐蔽）、锁阻塞、**主从延迟**
- ⭐ **`history list length`（`trx_rseg_history_len`）是最重要的监控指标**：> 10 万警告，> 100 万紧急
- ⭐ **头号排查 SQL**：`WHERE p.command = 'Sleep'` 且在 `INNODB_TRX` 里 → **空闲但开着事务**
- 预防措施：⚠️ **分批提交**、⚠️ **事务里不做外部调用**、只读用 autocommit、**应用设超时**
- 处理长事务：**先判断是不是正常业务**（`trx_query` 有 SQL 且时间在增长 = 等），⚠️ **KILL 之前想清楚回滚代价**
- 应急手段：扩容 undo 表空间（8.0 支持）、调大 `innodb_purge_batch_size`
- ⚠️ **原则：事务尽可能短**，但不要「每个语句一个事务」

## 练习

1. 长事务的两种形态是什么？哪一种更隐蔽？
2. `history list length` 是什么？多少算危险？为什么这是最重要的监控指标？
3. 为什么一个长事务提交后，数据库还会「不舒服」很久？
4. 写出「找出空闲但开着事务的连接」的 SQL，并解释每个条件的作用。
5. `KILL CONNECTION` 的代价是什么？什么情况下不该 KILL？
6. 「批量迁移数据」为什么不建议用一个大事务？分批的批大小怎么定？
7. 为什么说「在事务里调外部 API」是危险的？
8. `trx_rseg_history_len` 涨到 100 万，你的第一反应是什么？应急手段是什么？为什么那只是应急不是解法？

## 导航

[[mysql/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 上一篇：[[mysql/02-中段阶段-专题精讲/InnoDB存储引擎/专题概览]] · 下一篇：[[mysql/02-中段阶段-专题精讲/主从复制与高可用/复制原理与拓扑]]
