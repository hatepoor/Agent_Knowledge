---
tags: [mysql, 中段, 大表, DDL, pt-osc, gh-ost, 分区表, JSON]
type: guide
created: 2026-10-13
updated: 2026-10-13
---

# 大表运维与在线 DDL

> **什么时候该读**：表超过百万行要考虑；要做 `ALTER TABLE`；要清理历史数据；要用 JSON 处理多值字段。

## 学习目标

读完本篇，你应当能够：

1. 评估一张表「有多大算大」，以及为什么
2. 知道 MySQL 8.0 原生 `ALTER` 的能力边界
3. 在 `pt-online-schema-change` / `gh-ost` / 原生 DDL 之间做出选择
4. 安全地清理历史数据（分区表方案）
5. 判断什么时候该用 JSON、什么时候必须拆表
6. 掌握 `JSON_TABLE` 处理多值字段的用法

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[mysql/01-入门阶段-基础教程/第02章-建表与约束/2.1-DDL建表与列定义]] | `ALTER TABLE` 的代价 |

---

## 一、⚠️ 「大表」的阈值

⚠️ **没有一个绝对标准。** 关键看三个维度：

| 维度 | 说明 |
|------|------|
| **行数** | ⚠️ 主要因素 |
| ⭐ **数据体积** | ⭐ **比行数更重要**。⚠️ 1 亿行 × 10 字节 = 1GB；⚠️ 100 万行 × 10KB = 10GB |
| **索引数量** | ⚠️ 索引多，`ALTER` 时要重建更多次 |
| **访问频率** | ⚠️ 低频表（小字典、日志）慢点也能接受 |

### 参考阈值

| 表体积 | 影响 |
|--------|------|
| < 100 MB | ✅ 随便改 |
| 100 MB ~ 1 GB | ⚠️ 用原生 `ALTER`，选低峰期 |
| 1 GB ~ 10 GB | ⚠️⚠️ 原生 `ALTER` 要几十分钟，⚠️ 考虑 `gh-ost` |
| > 10 GB | ⚠️⚠️⚠️ **必须用在线 DDL 工具** |
| > 100 GB | ⚠️ 分区 + 归档，或考虑分库分表 |

⚠️ **⚠️ 别只看行数。** ⚠️ 一张 1000 万行但有 BLOB 大字段的表，比 1 亿行的窄表更难改。

---

## 二、原生 `ALTER` 的能力边界

### 8.0 的算法

| 算法 | 说明 | 8.0 支持 |
|------|------|---------|
| ⭐ `INSTANT` | 只改数据字典，⚠️ 几乎瞬间完成 | 8.0.12+ |
| `INPLACE` | ⭐ 不重建整表，但需要排序/重排 | ✅ 大部分操作 |
| `COPY` | ⚠️ 复制整表，⚠️ **阻塞写** | 少数操作 |
| `REORGANIZE` | 改变行存储顺序（只对部分引擎） | |

### 检查某个操作用什么算法

```sql
-- ⚠️ 关键习惯：改之前先看
EXPLAIN ALTER TABLE orders ADD COLUMN remark VARCHAR(200) NULL;
-- 输出：inplace / copy / instant
```

### 各操作的算法与代价

| 操作 | 8.0 算法 | 阻塞写 | 代价 |
|------|---------|--------|------|
| 加列（末尾，无默认值） | ⭐ `INSTANT` | ❌ | ⭐ 极低 |
| 加列（有默认值） | `INPLACE` | ❌ | 低 |
| 加列（中间位置） | `INPLACE` | ❌ | 中（要重排行） |
| 删列 | `INPLACE` | ❌ | 中 |
| 改列名 | ⭐ `INSTANT` | ❌ | 极低 |
| 改列类型（**扩大**） | `INPLACE` | ❌ | ⚠️ 有损转换不能 INPLACE |
| 改列类型（**缩小**） | ⚠️ `INPLACE` | ❌ | ⚠️ **可能数据丢失** |
| 加索引 | `INPLACE` | ❌ | ⚠️ 中（IO 密集） |
| 删索引 | `INPLACE` | ❌ | 低 |
| 改字符集 | `INPLACE` | ❌ | ⚠️ **重建整表，最高** |
| **改引擎** | ⚠️ `COPY` | ✅ **是** | ⚠️ 极高 |
| 加外键 | `INPLACE` | ❌ | 低 |
| ⚠️ **`OPTIMIZE TABLE`** | `COPY` | ⚠️ **重操作** | 极高 |

### ⚠️ `INSTANT` 的限制（8.0 时期）

| 限制 | 说明 |
|------|------|
| ⚠️ 加列位置 | ⚠️ 必须在**最后**（有默认值的只能紧跟 `AUTO_INCREMENT` 列后） |
| ⚠️ 不支持的操作 | 改列类型、加索引、删列、改字符集等 |
| ⚠️ 行格式 | 要求 `DYNAMIC` 或 `COMPRESSED`（⚠️ `COMPACT` 不行） |

```sql
-- 改行格式以便用 INSTANT
ALTER TABLE t ROW_FORMAT=DYNAMIC;   -- ⚠️ 这本身是重操作
```

⚠️ **⚠️ 8.0 的 `INSTANT` 限制较多**——⚠️ 后续版本（8.0.29+）放宽了。⚠️ **先 `EXPLAIN ALTER` 确认，别假设。**

### ⚠️「不阻塞写」≠「不慢」

⚠️ **`INPLACE` 期间会持续消耗 IO 和 CPU**，⚠️ 可能：
- 把主从延迟推高
- 让正常查询变慢
- 占满磁盘 IO

⚠️ **⚠️ 大表 `ALTER` 必须在低峰期做，并监控。**

### ⚠️ 显式指定算法（好习惯）

```sql
ALTER TABLE orders ADD COLUMN remark VARCHAR(200) NULL,
  ALGORITHM=INPLACE, LOCK=NONE;
```

⭐ **好处**：MySQL 发现做不到时**直接报错**，而不是悄悄降级到 `COPY` 锁表。

⚠️ **⚠️ `LOCK=NONE` 表示「不允许阻塞写」——如果做不到就报错。** ⭐ 这正是你要的：宁可失败也不要意外锁表。

---

## 三、两个在线 DDL 工具

### 机制（几乎相同）

```
① 创建影子表（新结构），命名为 _tablename_new
② 在影子上按块 INSERT 原表数据（每批几千行）
③ ⭐ 同步「期间发生的变更」：
   pt-osc → 触发器（AFTER INSERT/UPDATE/DELETE）
   gh-ost → binlog（ROW 格式）
④ 应用剩余变更
⑤ RENAME 原子交换（原表 → _old，影子表 → 原表）
⑥ 删掉旧表
```

### 对比

| 维度 | `pt-online-schema-change` | `gh-ost` |
|------|--------------------------|-----------|
| 原理 | ⚠️ **触发器** | ⭐ **binlog** |
| 主库额外开销 | ⚠️ ⚠️ 每个变更多 2 个触发器调用 | ⭐ 低（binlog 已有） |
| ⭐ 对复制的影响 | ⚠️ binlog 里会出现额外行 | ⭐ **干净**（binlog 无变化） |
| ⭐ 对延迟从库的影响 | ⚠️ ⚠️ **⚠️ 严重**（⚠️ binlog 膨胀 → 延迟暴增） | ⭐ 轻微 |
| 能否跨从库执行 | ✅ | ✅（⚠️ 需要从库权限） |
| 依赖 binlog | ❌ | ⭐ **必须 ROW 格式** |
| 触发器限制 | ⚠️ ⚠️ **⚠️ ⚠️ 表上有触发器就不能用** | ✅ 无此限制 |
| 流控 | `--max-load` | `--max-load` |
| 暂停/恢复 | ✅ `--max-load` 时暂停 | ⭐ `--throttle-control-replicas` 等 |

### ⭐ 关键结论

⚠️ **⚠️ 有延迟从库时，`gh-ost` 明显优于 `pt-osc`。**

⚠️ **⚠️ `pt-osc` 的致命问题**：⚠️ **它触发器产生的新变更会写进 binlog**，⚠️ **⚠️ 对一个 1 亿行的表做 pt-osc，可能产生数十 GB 的 binlog**，⚠️ **⚠️ 导致主从延迟数小时**。

⚠️ **⚠️ `gh-ost` 不动 binlog**，⚠️ 只在从库上读 binlog 并应用。⚠️ 主从延迟基本不受影响。

### `gh-ost` 常用命令

```bash
gh-ost \
  --alter="ENGINE=InnoDB" \
  --alter="ADD COLUMN remark VARCHAR(200) NULL, ALGORITHM=INSTANT" \
  --host=10.0.0.1 \
  --user=gh_ost \
  --password='xxx' \
  --database=shop \
  --table=orders \
  --allow-on-master \        # ⚠️ ⚠️ 需要：允许在主库上执行
  --max-load='Threads_running=30' \   # ⚠️ 流控：负载高时自动暂停
  --critical-load='Threads_running=60' \  # ⚠️ 超这个值就主动中断
  --concurrent-rowcount \    # ⚠️ 加速初次行数统计
  --exact-rowcount \         # ⚠️ 精确行数（配合 concurrent-rowcount）
  --assume-rbr \             # ⚠️ 允许用 STATEMENT 格式复制（⚠️ 慎用）
  --max-lag-millis=1500 \    # ⚠️ 从库延迟超过 1.5 秒就暂停
  --throttle-control-replicas='127.0.0.1:3306' \  # ⚠️ 控制所有从库
  --initially-drop-ghost-table \
  --initially-drop-old-table \
  --execute
```

⚠️ **关键参数**：

| 参数 | 说明 |
|------|------|
| ⭐ `--max-load` | ⭐ **流控**。主库负载超了就自动暂停 |
| ⭐ `--max-lag-millis` | ⭐ **从库延迟超了就暂停**（⚠️ 关键！） |
| ⚠️ `--allow-on-master` | ⚠️ ⚠️ **⚠️ 允许在主库上执行改表**——⚠️⚠️ **⚠️ 最危险的参数**。⚠️ 正常应该指定 `--host=<从库>` |
| `--concurrent-rowcount` | 加速初次行数统计 |
| `--cut-over=default` | ⚠️ 切换时机 |

### `pt-online-schema-change` 常用命令

```bash
pt-online-schema-change \
  --alter "ADD COLUMN remark VARCHAR(200) NULL, ALGORITHM=INPLACE, LOCK=NONE" \
  --host=10.0.0.2 \          # ⭐ 在从库上跑（减少对主库的影响）
  --user=root \
  --password='xxx' \
  D=shop,t=orders \
  --max-load='Threads_running=30' \
  --critical-load='Threads_running=60' \
  --chunk-size=1000 \        # 每批多少行
  --max-lag=1 \              # ⚠️ 从库延迟超过 1 秒暂停
  --check-interval=2 \
  --progress \
  --execute
```

⚠️ **⚠️ 相对 `gh-ost` 的三个限制**：

| 限制 | 说明 |
|------|------|
| ⚠️ **表上不能有触发器** | ⚠️ pt-osc 本身用触发器 |
| ⚠️ **binlog 必须开** | 它依赖 binlog 过滤变更 |
| ⚠️ ⚠️ **⚠️ 主从延迟影响大** | binlog 膨胀 |

### 选择决策

| 你的情况 | 选什么 |
|---------|--------|
| ⭐ **有延迟从库 / 对延迟敏感** | ⭐ **`gh-ost`** |
| 表上有触发器 | ⭐ `gh-ost`（pt-osc 不能用） |
| 表不大（< 10 GB）且低峰期 | 原生 `ALTER` 足够 |
| 只加列（8.0.12+） | ⭐ 原生 `INSTANT` |
| ⚠️ **要求对主库零影响** | 在从库上执行（⚠️ 注意：⚠️ 最后需要提升为主库时要 ⚠️ ⚠️ **重新做一次**，⚠️ 因为从库的 binlog 可能不完整） |

⚠️ **⚠️ 最后一条要注意**：⚠️ ⚠️ **在从库上改表，改完后从库表结构和主库不一致了**——⚠️ ⚠️ **如果之后要做主从切换，从库上没有这个变更**。⚠️ 正确做法是：**在从库上试跑验证 → 再在主库上执行**（⚠️ 生产主库上执行，用流控参数）。

---

## 四、清理历史数据

### ⚠️ 大批量 DELETE 的问题

```sql
-- ❌ 千万行的 DELETE
DELETE FROM logs WHERE created_at < '2025-01-01';
```

⚠️ **四个问题**：

| 问题 | 说明 |
|------|------|
| ⚠️ **巨大的 undo log** | 一千万行的 undo，⚠️ 磁盘和 purge 双重压力 |
| ⚠️ **长事务** | ⚠️ 执行期间就是长事务 |
| ⚠️ **主从延迟** | ⚠️ 从库要重放这个大事务 |
| ⚠️ **锁** | ⚠️ 长时间持锁 |

### 方案 1：分批 DELETE（最简单）

```sql
-- 循环执行，每批 10000 行
DELETE FROM logs WHERE created_at < '2025-01-01' LIMIT 10000;
-- 重复直到影响 0 行
```

⚠️ **⚠️ 每批之间要 sleep**，⚠️ 给 purge 和复制留时间：

```python
import time
while True:
    affected = delete_batch(10000)     # DELETE ... LIMIT 10000
    if affected == 0:
        break
    time.sleep(0.1)                    # ⭐ 给 IO 和复制留喘息
```

⭐ **⚠️ 这是没有分区表时的标准做法。** ⚠️ 批大小要根据实测调（⚠️ 从 1000 开始试）。

⚠️ **`DELETE ... LIMIT` 语法**：

```sql
DELETE FROM t WHERE condition ORDER BY id LIMIT 10000;   -- ⭐ 加 ORDER BY 保证顺序
-- ⚠️ 不加 ORDER BY 时，MySQL 选哪些行不确定，可能反复删同样的行
```

⚠️ **⚠️ 但注意：即使分批删，`.ibd` 文件大小不会变**——⚠️ 空间只是变成页内可复用。见 [[mysql/02-中段阶段-专题精讲/InnoDB存储引擎/专题概览]] 的膨胀章节。

### 方案 2：⭐ 分区表 + DROP PARTITION（最优）

```sql
-- ① 改造为分区表（⚠️ 本身是重操作，需要 gh-ost）
ALTER TABLE logs PARTITION BY RANGE (TO_DAYS(created_at)) (
  PARTITION p202501 VALUES LESS THAN (TO_DAYS('2025-02-01')),
  PARTITION p202502 VALUES LESS THAN (TO_DAYS('2025-03-01')),
  PARTITION p202503 VALUES LESS THAN (TO_DAYS('2025-04-01')),
  PARTITION pmax    VALUES LESS THAN MAXVALUE
);
```

⚠️ **⚠️ 分区表的三个硬性要求**：

| 要求 | 说明 |
|------|------|
| ⭐ **分区键必须包含在所有唯一索引里** | ⚠️ 否则 `ERROR 1503` |
| ⭐ **`TO_DAYS(created_at)` 必须是表达式的一部分** | ⚠️ 不能只写 `created_at`（它不是单调的） |
| ⚠️ **MySQL 只支持「水平分片」** | ⚠️ 每个分区是完整的数据子集 |

```sql
-- ② ⭐ 清理：秒级完成，空间立刻还给操作系统
ALTER TABLE logs DROP PARTITION p202501;
```

| 对比 | 分批 DELETE | DROP PARTITION |
|------|------------|----------------|
| 耗时 | ⚠️ 几十分钟到几小时 | ⭐ **秒级** |
| undo 压力 | ⚠️ 巨大 | ⭐ **几乎为零** |
| 主从延迟 | ⚠️ 严重 | ⭐ **很小**（⚠️ 8.0 后更小） |
| ⭐ 空间还给系统 | ❌ 否 | ⭐ **是** |
| 可恢复 | ⚠️ 不能 | ⭐ **⚠️ 不能（但可以先备份该分区）** |

⭐ **⚠️ 重要提醒**：`DROP PARTITION` 是**不可逆的**。⚠️ **执行前先 `ALTER TABLE ... REMOVE PARTITIONING`？不——应该先导出该分区数据备份。**

⚠️ **MySQL 8.0 的改进**：⚠️ **`DROP PARTITION` 在 binlog 里是轻量事件**（⚠️ 8.0 之前会记录每行的删除，⚠️ ⚠️ 导致 binlog 暴涨）——⚠️ 这也是 8.0 更适合分区表清理的原因。

### 方案 3：归档到冷表

```sql
-- ① 创建归档表（同结构，无索引）
CREATE TABLE logs_archive LIKE logs;

-- ② 分批搬
INSERT INTO logs_archive SELECT * FROM logs
WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31' LIMIT 10000;
-- 重复

-- ③ 确认搬完后删除
DELETE FROM logs WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31';
```

⚠️ **⚠️ 「INSERT 后 DELETE」不是原子的**——⚠️ 中途崩溃会有重复。⚠️ 需要用事务或对账。

---

## 五、JSON 与多值字段建模

### ⚠️ 核心决策

| 需求 | 方案 |
|------|------|
| 属性不固定、只在本行用、不参与 JOIN | ⭐ **JSON** |
| 多值且需要按这个值 JOIN 别的表 | ⭐ **关联表** |
| 多值且要频繁查询 | ⚠️ JSON + `JSON_TABLE`（⚠️ 但无法建索引） |
| ⚠️ 需要外键保证一致 | ⭐ **关联表**（JSON 无外键） |

### ⚠️ 三种方案的详细对比

#### 方案 A：JSON 列

```sql
CREATE TABLE products (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(64),
  tags JSON                      -- ["electronics","hot"]
);
```

| 优点 | 缺点 |
|------|------|
| ✅ 不用维护关联表 | ❌ **不能建索引** |
| ✅ 结构灵活 | ❌ **不能有外键** |
| ✅ 读取方便 | ❌ **⚠️ 统计和分页很痛** |
| | ❌ **⚠️ 大数据量下 JOIN 展开开销大** |

⚠️ **⚠️ JSON 的致命限制：不能建普通索引。** 解法是生成列：

```sql
ALTER TABLE products ADD COLUMN tag_count INT
  GENERATED ALWAYS AS (JSON_LENGTH(tags)) STORED;
CREATE INDEX idx_tag_count ON products (tag_count);

-- 按单个 tag 查（用生成列）
ALTER TABLE products ADD COLUMN primary_tag VARCHAR(50)
  GENERATED ALWAYS AS (tags->>'$[0]') STORED;
CREATE INDEX idx_primary_tag ON products (primary_tag);
```

⚠️ **⚠️ 生成列只能索引 JSON 的「固定路径」**，⚠️ `tags->>'$[*]'`（所有元素）**无法索引**。

#### 方案 B：⭐ 关联表（标准做法）

```sql
CREATE TABLE products (id BIGINT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(64));

CREATE TABLE product_tags (
  product_id BIGINT NOT NULL,
  tag VARCHAR(50) NOT NULL,
  PRIMARY KEY (product_id, tag),
  KEY idx_tag (tag)                    -- ⭐ 可以按 tag 反查
) ENGINE=InnoDB;

-- 关联
INSERT INTO product_tags VALUES (1, 'electronics'), (1, 'hot');

-- 查询
SELECT p.* FROM products p
JOIN product_tags pt ON pt.product_id = p.id
WHERE pt.tag = 'hot';                   -- ⭐ 走 idx_tag
```

| 优点 | 缺点 |
|------|------|
| ⭐ **能建索引** | ❌ 多一张表 |
| ⭐ **能加外键** | ❌ 多一次 JOIN |
| ⭐ **能保证一致** | |

⭐ **⚠️ 决策很简单：会查询/统计/排序的多值字段，一律用关联表。**

#### 方案 C：`JSON_TABLE` 展开

⚠️ **这是「已经用了 JSON」的补救方案。**

```sql
CREATE TABLE products (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(64),
  tags JSON
);

INSERT INTO products VALUES
  (1, '手机', '["electronics","hot","sale"]'),
  (2, '耳机', '["electronics"]');
```

```sql
-- ⭐ 把 JSON 数组展开成行
SELECT p.id, p.name, t.tag
FROM products p
JOIN JSON_TABLE(
  p.tags,                           -- 源数组
  '$[*]' COLUMNS (tag VARCHAR(50) PATH '$')
) AS t;
```

结果：

```
id | name | tag
1  | 手机  | electronics
1  | 手机  | hot
1  | 手机  | sale
2  | 耳机  | electronics
```

⚠️ **⚠️ 三个限制**：

| 限制 | 说明 |
|------|------|
| ⚠️ **每次查询都要展开** | 无法索引，每次都算 |
| ⚠️ **数据量大时开销大** | ⚠️ 10 万行 × 5 个标签 = 50 万次展开 |
| ⚠️ **不能有外键** | 一致性要靠应用 |

⭐ **所以 `JSON_TABLE` 适合「临时查询/导出」，不适合高频查询。**

### 决策树

```
这个多值字段需要：
├── 按它 JOIN 别的表？        → 是 → 关联表
├── 按它高频查询/筛选？        → 是 → 关联表
├── 按它排序/统计？            → 是 → 关联表
├── 需要外键保证一致？         → 是 → 关联表
└── 只是存下来偶尔整体读？    → 否 → ⭐ JSON 可以
```

⚠️ **⚠️ 订单明细这类数据绝对不能用 JSON**：⚠️ 需要按商品统计、需要和商品表 JOIN、需要单条退款操作——⚠️ 全都需要独立行。

---

## 六、其他大表运维

### 大表加索引

⚠️ **⚠️ 千万行表加索引要预估时间**：

```sql
-- 看索引大小估算
SELECT (AVG_ROW_LENGTH * TABLE_ROWS) / 1024 / 1024 AS data_mb
FROM information_schema.TABLES WHERE TABLE_NAME = 'orders';

-- 看现有索引大小
SELECT INDEX_NAME,
       ROUND((STATIC_INDEX_SIZE/1024/1024)) AS stat_mb,
       ROUND((NON_UNIQUE_INDEX_SIZE/1024/1024)) AS non_uniq_mb
FROM mysql.innodb_table_stats WHERE table_name = 'orders';
```

⚠️ **`innodb_table_stats` 是采样统计，可能不准**，⚠️ 但足够做量级判断。

⚠️ **大表加索引**：⚠️ 原生 `ALTER ... ADD INDEX` 是 `INPLACE` 不阻塞写，⚠️ 但 ⚠️ **IO 消耗大，可能影响正常查询**。⚠️ 建议低峰期 + 用 `gh-ost`（⚠️ 但它对「加索引」的操作不太适用，⚠️ 原生就够）。

### 大表清理归档的整体流程

```
① 确认要清理的数据范围和量
        ↓
② ⭐ 分区表？→ 改造为分区表（gh-ost）→ DROP PARTITION（秒级）
   不是？→ 分批 DELETE + sleep
        ↓
③ 监控：undo、history list length、主从延迟
        ↓
④ 确认完成：SELECT COUNT(*) 对比预期
        ↓
⑤ ⭐ 验证空间：SELECT SLEEP(0); OPTIMIZE TABLE（如果需要缩文件）
```

⚠️ **`OPTIMIZE TABLE` 是最后手段**：⚠️ 它重建整表，⚠️ 千万行要几十分钟。⚠️ **用 `gh-ost` 做**（⚠️ 或者用「分区表 + 重建」）。

---

## 七、陷阱

1. **❌ 在高峰期对大表做 `ALTER`。** ⚠️ 即使 `INPLACE` 也会拖慢 IO。
2. **❌ 不加 `ALGORITHM=INPLACE, LOCK=NONE`。** ⚠️ 可能悄悄降级到 COPY 锁表。
3. **❌ 假设 8.0 的 `INSTANT` 支持所有加列场景。** ⚠️ 先 `EXPLAIN ALTER`。
4. **⚠️ ⚠️ 用 `pt-osc` 导致主从延迟数小时。** ⭐ **有延迟从库时用 `gh-ost`**。
5. **⚠️ ⚠️ 在主库上执行 `gh-ost` 忘了 `--allow-on-master`。** ⚠️ 或者更糟：⚠️ ⚠️ **不加这个参数直接在从库执行但之后要切主**——⚠️ 从库没有这个变更。
6. **❌ `gh-ost` 不设 `--max-lag-millis`。** ⚠️ 改表把从库延迟推到几小时。
7. **❌ 一次性 `DELETE` 千万行。** ⚠️ undo 爆炸 + 长事务 + 主从延迟。
8. **❌ 分批 DELETE 不加 `ORDER BY`。** ⚠️ 可能反复删同样的行。
9. **❌ 在从库上 `pt-osc` 不考虑主从切换。** ⚠️ ⚠️ 从库上没有变更记录。
10. **❌ 多值字段用 JSON 而需要高频查询。** ⭐ 用关联表。
11. **❌ `OPTIMIZE TABLE` 用在大表上。** ⚠️ 用 `gh-ost` 或分区重建。

---

## 小结

- ⚠️ **「大表」要看数据体积而非行数**，索引数量和访问频率也要考虑
- ⚠️ **改之前先 `EXPLAIN ALTER`** 看用什么算法
- **8.0 的 `INSTANT`** 几乎瞬间，但⚠️ 限制较多（加列位置、只改字典等）
- ⭐ **必须加 `ALGORITHM=INPLACE, LOCK=NONE`**——宁可失败也不要悄悄降级锁表
- ⚠️ **「不阻塞写」≠「不慢」**——INPLACE 期间会耗 IO 和 CPU
- ⭐ **`gh-ost` 优于 `pt-osc` 的核心原因：不动 binlog**，⚠️ 对延迟从库友好得多
- ⚠️ **`pt-osc` 在亿行表上会产生数十 GB binlog**，⚠️ 导致从库延迟数小时
- `gh-ost` 关键参数：⭐ `--max-load`（流控）、⭐ `--max-lag-millis`（从库延迟暂停）
- 清理历史数据：⭐ **分区表 + `DROP PARTITION` 是最优**（秒级 + 空间还给系统）；没有分区表就**分批 DELETE + `ORDER BY` + sleep**
- ⚠️ MySQL 分区表要求：**分区键必须在所有唯一索引里**
- JSON 多值字段：⭐ **会查询/统计/JOIN 的一律用关联表**；JSON 只适合「存下来偶尔整体读」

## 练习

1. 「大表」应该看什么维度？为什么行数不是唯一标准？
2. 改表之前应该先执行什么命令？为什么？
3. 为什么建议给 `ALTER TABLE` 加 `ALGORITHM=INPLACE, LOCK=NONE`？
4. `gh-ost` 和 `pt-online-schema-change` 的核心机制差异是什么？为什么有延迟从库时前者更好？
5. `gh-ost` 的 `--max-load` 和 `--max-lag-millis` 各起什么作用？
6. 清理 5000 万行历史数据，给出两种方案并对比。
7. 分区表建分区时，对唯一索引有什么要求？
8. 「删除 1000 万行」为什么会让 `.ibd` 文件不变小？怎么彻底解决？
9. 一个商品有多个标签，用户经常「按标签筛商品」，用 JSON 还是关联表？为什么？
10. `JSON_TABLE` 展开的三个限制是什么？

## 导航

[[mysql/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 上一篇：[[mysql/02-中段阶段-专题精讲/工具与可观测性/工具链总览]] · 进阶阶段：[[mysql/03-进阶阶段-深度剖析/00-剖析方法论|剖析方法论]] · 返回：[[mysql/学习路线图|学习路线图]]
