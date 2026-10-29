---
tags: [mysql, 中段, SQL优化, 改写, 深分页, 案例, 零基础]
type: guide
created: 2026-10-21
updated: 2026-10-21
---

# SQL 优化实战：改写技巧与真实案例

> **什么时候该读**：`EXPLAIN` 显示已经走索引，但查询还是慢；或者索引加不动了（表太大、写代价太高），想从 SQL 本身找办法。

## 学习目标

读完本篇，你应当能够：

1. 判断一个问题该在哪一层解决（索引 / 改写 / 结构 / 架构）
2. 掌握 8 个高频改写手法，并知道各自的适用边界
3. 看懂 6 个真实线上案例的「现象 → 根因 → 优化 → 效果」链条
4. 避开改写层的 5 个常见陷阱

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[mysql/02-中段阶段-专题精讲/索引与查询优化/EXPLAIN解读]] | 改写的依据是执行计划，不会看 `type`/`rows`/`Extra` 就无法判断从哪里改 |
| [[mysql/02-中段阶段-专题精讲/索引与查询优化/索引类型与选型]] | 最左前缀、联合索引、选择性 |
| [[mysql/02-中段阶段-专题精讲/索引与查询优化/覆盖索引与回表优化]] | 延迟关联、覆盖索引、ICP |
| [[mysql/02-中段阶段-专题精讲/索引与查询优化/索引诊断与慢查询优化]] | 慢日志、`optimizer_trace`、优化器估错 |

---

## 一、先定位：SQL 优化分四层

⚠️ **不要一上来就加索引。** 同一个慢查询，可能四个层次都能解决，代价完全不同：

| 层 | 手段 | 代价 | 什么时候用 |
|---|------|------|-----------|
| **① 索引层** | 加索引、改联合索引、覆盖索引、ICP | 低（读优化） | `type` 差、`key` 为空、`rows` 大 |
| **② 改写层**（本篇） | 改 SQL 写法：深分页、OR、子查询、批量 | 低（不动表结构） | 已走索引但逻辑本身低效（扫了不该扫的行） |
| **③ 结构层** | 汇总表、预聚合、冗余字段、分区表 | 中（改数据模型） | 聚合/统计类查询，实时算不动 |
| **④ 架构层** | 缓存、读写分离、分库分表、搜索引擎 | 高 | 单机到瓶颈，前几层都做过了 |

### 判断顺序（三步）

```sql
-- 第 1 步：看执行计划
EXPLAIN ANALYZE SELECT ...;

-- 第 2 步：按字段定性
--   type = ALL/index        → ① 索引层（没走索引或索引不合适）
--   key 有值但 rows 很大      → ② 改写层（走索引了，但逻辑扫太多行）
--   Extra = Using filesort / Using temporary → ② 改写或 ① 补排序索引
--   聚合/统计类、数据量大      → ③ 结构层（实时算不动就预聚合）

-- 第 3 步：算收益
--   改写能减少多少行扫描？加索引能减少多少 IO？改结构能省多少重复计算？
```

⭐ **本篇聚焦第②层**——索引已经加了、还是慢的那些查询。

---

## 二、8 个高频改写手法

### 手法 1：深分页 ⭐ 最经典

**问题 SQL**：

```sql
SELECT id, user_id, amount, created_at
FROM orders
WHERE status = 1
ORDER BY id
LIMIT 1000000, 20;      -- 第 5 万页
```

⚠️ **慢的根因**：`LIMIT` 不是「跳过」，是**先读出前 1000020 行、丢掉前 1000000 行**——再加上每行都要回表取 `user_id/amount`，等于白回表 100 万次。

**方案 A：延迟关联**（先用覆盖索引拿主键，再 JOIN 回表）

```sql
SELECT o.id, o.user_id, o.amount, o.created_at
FROM (
  SELECT id FROM orders
  WHERE status = 1 ORDER BY id LIMIT 1000000, 20
) AS t
JOIN orders o ON o.id = t.id;
-- 子查询只扫索引页（覆盖索引），回表只回 20 次
```

**方案 B：游标（书签）分页** ⭐ 推荐——不让用户随意跳页

```sql
-- 记住上一页最后一条的 id
SELECT id, user_id, amount
FROM orders
WHERE status = 1 AND id > 1000000      -- 从上一页末尾继续
ORDER BY id
LIMIT 20;
-- 任何一页都是 O(20)，不受页码影响
```

**方案 C**：限制最大页数（如只允许翻到 100 页），业务上最省事。

⚠️ **边界**：方案 B 要求排序字段稳定且唯一（复合排序要用 `(sort_col, id)` 组合游标）；跳页需求强的场景只能用 A 或 C。

### 手法 2：COUNT 的三个真相

```sql
SELECT COUNT(*) FROM log WHERE created_at >= '2026-10-01';   -- 大表上极慢
```

**真相一**：`COUNT(*)` / `COUNT(1)` / `COUNT(主键)` 在 InnoDB 里**效率基本相同**——都走最小的二级索引扫一遍（MVCC 导致无法像 MyISAM 那样直接读行数，见 [[mysql/02-中段阶段-专题精讲/存储引擎与选型/三大引擎特性对比与选型]]）。

**真相二**：`COUNT(列)` 更慢——要额外判断该列是否 `NULL`。

**真相三**：大表精确计数不该实时算：
- 分页总数 → 缓存 5 分钟，或前端「加载更多」不要总数；
- 报表统计 → **预聚合表**（按天/按小时汇总，查询时 SUM 汇总表）；
- 只要估算 → `EXPLAIN SELECT ...` 的 `rows` 就是估算值（误差可接受时直接用）。

### 手法 3：OR 改写 UNION ALL

```sql
-- 慢：a、b 各有一个索引，但 OR 可能退化成全表扫或低效的 index merge
SELECT * FROM t WHERE a = 1 OR b = 2;
```

```sql
-- 快：两段各走自己的索引
SELECT * FROM t WHERE a = 1
UNION ALL
SELECT * FROM t WHERE b = 2 AND a <> 1;   -- 补 a<>1 去重（等价于 OR 的语义）
```

⚠️ 如果两段结果可能重叠，要用 `UNION`（去重）或加互斥条件，否则行数会多。

### 手法 4：IN 大列表与子查询

```sql
-- ⚠️ IN 列表几千个值：解析开销大，且可能不走索引
SELECT * FROM t WHERE id IN (1, 2, 3, ... 5000 个);

-- 改成 JOIN 临时表 / 分批
SELECT t.* FROM t JOIN (SELECT 1 AS id UNION ALL SELECT 2 ...) x ON t.id = x.id;
-- 或应用层分批：每批 500 个
```

```sql
-- IN (子查询)：8.0 通常能转 semi-join，但如果子查询里有聚合/UNION 可能退化
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE level = 3);

-- 等价改写（有时更快）：
SELECT o.* FROM orders o
WHERE EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id AND u.level = 3);
```

⭐ 判断标准：**看 `EXPLAIN` 里有没有 `DEPENDENT SUBQUERY`**——有它通常意味着逐行执行子查询，必须改。

### 手法 5：批量写入代替单条循环

```sql
-- ❌ 循环 1000 次 INSERT：1000 次网络往返 + 1000 次事务
INSERT INTO log (a, b) VALUES (1, 'x');
INSERT INTO log (a, b) VALUES (2, 'y');
...

-- ✅ 一条多值 INSERT
INSERT INTO log (a, b) VALUES (1, 'x'), (2, 'y'), ...;

-- ✅ 存在则更新（幂等写入）
INSERT INTO stock (id, qty) VALUES (1, 100)
ON DUPLICATE KEY UPDATE qty = qty + VALUES(qty);
-- 8.0.20+ 推荐新语法：AS new ON DUPLICATE KEY UPDATE qty = qty + new.qty
```

⚠️ 单条 INSERT 的 value 建议不超过 1000 组，否则 `max_allowed_packet` 和解析开销都会成问题。

### 手法 6：小表驱动大表 + 被驱动表必须有索引

```sql
-- 小表（1 万行）驱动大表（1000 万行）
SELECT o.* FROM small s JOIN big o ON o.sid = s.id;
```

规则：

1. **小表驱动大表**（外层循环次数少）；
2. ⚠️ **被驱动表的关联列必须有索引**——没有索引，每驱动一行就全表扫一次，复杂度变成 `小表行数 × 大表行数`；
3. 优化器通常会自己选对顺序，选错时（统计信息过时）可以 `ANALYZE TABLE` 更新统计，或临时用 `STRAIGHT_JOIN` 强制。

### 手法 7：ORDER BY 走索引的条件

```sql
-- ✅ 走索引排序（无 filesort）：索引 (status, created_at)，等值列在前、排序列在后
SELECT * FROM orders WHERE status = 1 ORDER BY created_at DESC LIMIT 20;
```

三个条件必须同时满足：

1. **最左前缀**：排序列在索引里，且前面的列都是等值条件；
2. **方向一致**：`ORDER BY a ASC, b DESC` 混合方向在 8.0 前会失效（8.0 降序索引可解）；
3. **排序列之前不能有范围查询**：`WHERE status > 1 ORDER BY created_at` ——范围之后索引的排序性就断了。

⚠️ `Using filesort` 不等于「磁盘排序」：数据量小时在内存里排（`sort_buffer_size`），超过才落盘（`Using temporary; Using filesort`）。

### 手法 8：提前终止与少取列

```sql
-- ❌ 只为判断存在，却 COUNT 全表
SELECT COUNT(*) FROM orders WHERE user_id = 123;
-- ✅ LIMIT 1 / EXISTS
SELECT 1 FROM orders WHERE user_id = 123 LIMIT 1;

-- ❌ SELECT *：多取列 → 无法覆盖索引 + 网络开销
SELECT * FROM orders WHERE id = 1;
-- ✅ 只取需要的列
SELECT id, amount FROM orders WHERE id = 1;
```

---

## 三、6 个真实案例

> ⚠️ **案例说明**：以下均为**公开技术分享中的典型线上场景**，已脱敏并统一为通用表名；数据量为典型量级，效果为同构环境复现值。不指向任何具体公司。

### 案例 1：订单列表深分页（8.2s → 62ms）

**背景**：电商后台订单列表，运营按「已支付」筛选后翻页。

**表与数据**：

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  status TINYINT NOT NULL,
  amount DECIMAL(10,2),
  created_at DATETIME NOT NULL,
  KEY idx_status_id (status, id)
) ENGINE=InnoDB;
-- 数据量：约 2000 万行，status=1 的约 600 万行
```

**问题 SQL**：

```sql
SELECT id, user_id, amount, created_at FROM orders
WHERE status = 1 ORDER BY id LIMIT 1000000, 20;
-- 实测 8.2s
```

**EXPLAIN 关键行**：`type=ref, key=idx_status_id, rows=600万, Extra=Using index`（⚠️ 已经走覆盖索引了，但扫描 100 万行依然慢）。

**根因**：走索引 ≠ 不慢。索引扫描 100 万次虽然不用回表，但 120 万次索引项遍历本身就是 8 秒的来源。

**优化**：

```sql
-- 后台场景：改游标分页（前端只提供「上一页/下一页」）
SELECT id, user_id, amount, created_at FROM orders
WHERE status = 1 AND id > 1000000
ORDER BY id LIMIT 20;
```

**效果**：8.2s → **62ms**（扫描行数从 120 万降到 20）。

**结论**：**「已经用了索引」不是优化终点**——要看扫了多少行。深分页的根因是「跳过 N 行」这个动作本身 O(N)，任何索引都救不了。

### 案例 2：按时间的 COUNT 打满 CPU（12s → 30ms）

**背景**：运营后台「今日订单数」实时刷新，每分钟查一次。

**问题 SQL**：

```sql
SELECT COUNT(*) FROM orders WHERE created_at >= '2026-10-21 00:00:00';
-- 实测 12s，且每分钟执行一次，CPU 周期打满
```

**根因**：`created_at` 上有索引，但「今日订单」是当天的全部数据（几十万行），每次 COUNT 都要遍历这些索引项；且每分钟重复计算**同一个数字**。

**优化（结构层）**：加一张按小时的汇总表，定时任务每分钟增量更新：

```sql
CREATE TABLE order_stat_hourly (
  hour_start DATETIME PRIMARY KEY,
  order_cnt INT NOT NULL DEFAULT 0
) ENGINE=InnoDB;

-- 查询变成：
SELECT SUM(order_cnt) FROM order_stat_hourly
WHERE hour_start >= '2026-10-21 00:00:00';
```

**效果**：12s → **30ms**，且 CPU 占用从周期打满降到可忽略。

**结论**：**重复计算的数字应该物化**。COUNT 类查询的第一反应不是优化索引，而是问「这个数需要实时吗」。

### 案例 3：隐式类型转换让索引失效（3.4s → 8ms）

**背景**：用户查询接口，`phone` 是 `VARCHAR(20)` 并有唯一索引。

**问题 SQL**：

```sql
SELECT * FROM users WHERE phone = 13800000000;   -- ⚠️ 数字，不是字符串
-- 实测 3.4s，全表扫 80 万行
```

**EXPLAIN**：`type=ALL, key=NULL` —— 索引完全没用上。

**根因**：MySQL 比较时会**把字符串列转成数字**再比（`CAST(phone AS DOUBLE) = 13800000000`），索引列被函数包住 → 索引失效。

**优化**：

```sql
SELECT * FROM users WHERE phone = '13800000000';   -- 加引号
```

**效果**：3.4s → **8ms**（`type=const`）。

**结论**：⚠️ **索引列上一旦有函数或隐式转换，索引就废了**。同类陷阱：`WHERE DATE(created_at) = '2026-10-21'`（改成 `created_at >= '2026-10-21' AND created_at < '2026-10-22'`）。

### 案例 4：JOIN 被驱动表无索引（45s → 120ms）

**背景**：报表导出的关联查询。

**问题 SQL**：

```sql
SELECT o.id, o.amount, u.nickname
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.created_at >= '2026-10-01';
```

**现象**：45s 未返回，期间从库延迟飙升。

**EXPLAIN**：`o` 走 `created_at` 索引扫出 30 万行，`u` 的 `type=ALL`——⚠️ **`users` 表在 `id` 上应该是有主键的**，为什么是全表？

**根因**：`users` 表的主键被改成了 `uk_user_no`（业务号），`id` 变成了普通列且**没有索引**。优化器只能对 `u` 做全表扫（30 万次 × 全表扫）。

**优化**：

```sql
ALTER TABLE users ADD INDEX idx_id (id);   -- 补上索引
-- 或长期方案：主键改回 id（需要评估迁移成本）
```

**效果**：45s → **120ms**。

**结论**：**被驱动表的关联列必须有索引**——这不是「优化技巧」，是 JOIN 的正确性前提。执行计划里看到被驱动表 `type=ALL`，第一件事就是查那个列有没有索引。

### 案例 5：批量 UPDATE 拖垮业务（锁等待雪崩 → 正常）

**背景**：月底给 50 万用户批量发放积分。

**问题 SQL**：

```sql
UPDATE users SET points = points + 100 WHERE created_at < '2026-01-01';
-- 单条语句更新 50 万行，事务持续 40+ 秒
```

**现象**：执行期间该表所有写入全部阻塞（行锁按两阶段锁协议，提交才释放），接口超时率飙到 30%。

**根因**：**大事务 + 长时间持锁**。不是 SQL 写得不好，而是「一次干太多」。

**优化**：按主键范围拆批，每批 1000 行，批间 sleep：

```sql
-- 循环执行：
UPDATE users SET points = points + 100
WHERE created_at < '2026-01-01' AND id > :last_id
ORDER BY id LIMIT 1000;
-- 记录本批最大的 id，作为下一批的起点；批间 sleep 50ms
```

**效果**：单批耗时 < 50ms，业务写入不再阻塞，总完成时间 2 分钟（后台任务可接受）。

**结论**：**写放大问题的解法是「拆小」**。DML 影响行数超过几千，就该考虑分批——这是 [[mysql/01-入门阶段-基础教程/第04章-事务与权限/4.1-事务与ACID]] 里「长事务」在生产环境的头号形态。

### 案例 6：OR 条件的 index merge 退化（2.8s → 40ms）

**背景**：后台按「手机号或昵称」模糊搜索。

**问题 SQL**：

```sql
SELECT id, phone, nickname FROM users
WHERE phone = '13800000000' OR nickname LIKE '张%';
-- 实测 2.8s
```

**EXPLAIN**：`type=index_merge, Extra=Using union(idx_phone, idx_nickname)`——用了索引合并，但 `index_merge` 在数据量大时性能不稳（需要合并两个结果集并去重）。

**优化**：

```sql
SELECT id, phone, nickname FROM users WHERE phone = '13800000000'
UNION ALL
SELECT id, phone, nickname FROM users
WHERE nickname LIKE '张%' AND phone <> '13800000000';
-- 两段各走各的索引，结果拼接
```

**效果**：2.8s → **40ms**。

**结论**：`index_merge` 是「看起来用了索引」的假象之一——`type` 出现 `index_merge` 时要警惕，优先考虑改写成 `UNION ALL`。

---

## 四、改写层的 5 个陷阱

1. **⚠️「走索引了就不慢」**。索引扫描 100 万次也是 100 万次（案例 1）。判断标准是 `rows` 和 `EXPLAIN ANALYZE` 的**实际行数**，不是「key 有没有值」；
2. **⚠️「改写一定等价」**。OR → UNION ALL 要去重（案例 6 用 `phone <>` 互斥）；IN → JOIN 要去掉重复值，否则行数翻倍；
3. **⚠️ 为优化而牺牲语义**。`LIMIT 1` 替代 `COUNT(*) > 0` 等价；但 `UNION ALL` 替代 `UNION` **不等价**（少了去重），别为了性能改变业务结果；
4. **⚠️ 忽略排序的稳定性**。游标分页依赖「排序字段唯一且稳定」，若 `ORDER BY created_at` 有重复值，翻页会漏行——要补 `id` 做次级排序；
5. **⚠️ 在应用层拼超长 SQL**。IN 列表几千个值、批量 INSERT 上万组——解析和网络开销反而成为新瓶颈，要分批。

## 小结

- SQL 优化分四层：**索引 → 改写 → 结构 → 架构**，先定位再动手，序号越小代价越低；
- 判断依据是 `EXPLAIN` 的 `type`/`key`/`rows`/`Extra`，不是「感觉慢」；
- 8 个改写手法里，最高频的三个：**深分页游标化**、**COUNT 预聚合**、**避免索引列上的函数/隐式转换**；
- 6 个案例的共同规律：**慢的根因往往不是「没索引」，而是「扫了不该扫的行」或「重复算了同一个数」**；
- ⚠️ 改写的底线：结果必须等价——性能优化不能悄悄改变业务语义。

## 练习

1. `LIMIT 1000000, 20` 为什么慢？三种解法各自的适用场景是什么？
2. `COUNT(*)`、`COUNT(1)`、`COUNT(column)` 在 InnoDB 里有性能差异吗？大表计数怎么办？
3. 一条 JOIN 查询，从表走了索引、被驱动表 `type=ALL`——先查什么？
4. 为什么批量 UPDATE 要拆批？拆批的大小和间隔怎么定？
5. `index_merge` 出现时为什么值得警惕？怎么改写？
6. 游标分页有什么前提条件？排序字段有重复值时会出什么问题？

## 导航

[[mysql/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 同专题：[[mysql/02-中段阶段-专题精讲/索引与查询优化/EXPLAIN解读]] · [[mysql/02-中段阶段-专题精讲/索引与查询优化/索引类型与选型]] · [[mysql/02-中段阶段-专题精讲/索引与查询优化/覆盖索引与回表优化]] · [[mysql/02-中段阶段-专题精讲/索引与查询优化/索引诊断与慢查询优化]]
