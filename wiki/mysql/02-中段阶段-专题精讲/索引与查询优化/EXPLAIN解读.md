---
tags: [mysql, 中段, EXPLAIN, 执行计划, 性能]
type: guide
created: 2026-10-13
updated: 2026-10-13
---

# EXPLAIN 解读

> **什么时候该读**：任何「查询慢」的问题，**第一步就是 EXPLAIN**。

## 学习目标

读完本篇，你应当能够：

1. 逐字段读懂 `EXPLAIN` 的输出
2. 判断 `type` 字段的优劣，知道哪些值意味着问题
3. 解释 `Extra` 字段里每个关键字的含义
4. 用 `EXPLAIN ANALYZE` 拿到真实耗时
5. 走完「EXPLAIN → 定位问题 → 优化 → 验证」的完整流程
6. 知道什么时候 `type=ALL` 是正常的（小表）

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[mysql/02-中段阶段-专题精讲/索引与查询优化/索引类型与选型]] | 最左前缀、索引组织 |

---

## 一、先执行一遍

### 传统格式

```sql
EXPLAIN
SELECT e.name, d.name AS dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id
WHERE e.status = 1
ORDER BY e.created_at DESC
LIMIT 20;
```

输出：

```
+----+-------------+-------+------------+-------+---------------+-------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key   | key_len |
+----+-------------+-------+------------+-------+---------------+-------+-------------+
|  1 | SIMPLE      | e     | NULL       | ref   | idx_status    | idx_status | 2 |
|  1 | SIMPLE      | d     | NULL       | eq_ref| PRIMARY       | PRIMARY    | 4 |
+----+-------------+-------+------------+-------+---------------+-------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key   | key_len | ref   | rows   | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+-------+---------+-------+---------+---------+
|  1 | SIMPLE      | e     | NULL       | ref   | idx_status    | idx_status | 2  | const | 12345  | 100.00   | Using where; Using index |
|  1 | SIMPLE      | d     | NULL       | eq_ref| PRIMARY       | PRIMARY    | 4  | const | 1     | 100.00   | NULL     |
+----+-------------+-------+------------+-------+---------------+-------+---------+-------+---------+---------+
```

### 树状格式（更易读）

```sql
EXPLAIN FORMAT=TREE
SELECT ...;
```

⭐ **树状格式在 8.0.18+ 最好用**，它按「实际执行顺序」展示，并标出每层的耗时。

### `EXPLAIN ANALYZE`（8.0.18+）⭐

```sql
EXPLAIN ANALYZE SELECT ...;
```

⚠️ **⚠️ 它会真的执行这条语句！**

| 风险 | 说明 |
|------|------|
| ⚠️ `INSERT`/`UPDATE`/`DELETE` **会真的改数据** | ⚠️ **绝对不能对写语句直接跑** |
| ⚠️ 耗时是**真实的**（不是估算） | ⭐ 这是它最大的价值 |
| ⚠️ 会实际消耗资源 | 在生产上跑要谨慎 |

⭐ **正确用法**：在测试环境跑，或把写语句改成对应的 `SELECT` 再跑。

输出示例：

```
-> Limit: 20 row(s)  (actual time=0.123..0.210 rows=20 loops=1)
    -> Nested loop inner join  (cost=1234.5 rows=12345) (actual time=0.100..0.195 rows=20 loops=1)
        -> Index lookup on employees using idx_status (status=1)  (cost=0.5 rows=12345) (actual time=0.080..0.150 rows=20 loops=1)
        -> Single-row index lookup on departments using PRIMARY (id=in(...))  (cost=0.3 rows=1) (actual time=0.010..0.015 rows=1 loops=20)
```

⭐ **`actual time=0.080..0.195`** 才是真实的行处理耗时，**`cost=1234.5` 只是估算**。**估算和真实差一个数量级，就说明优化器估错了行数。**

---

## 二、`type` 字段：从好到坏

⚠️ **`type` 是判断索引是否生效的第一指标。**

| `type` | 名称 | 说明 | 评价 |
|--------|------|------|------|
| `system` | 系统表 | 只有一行（`dual` 表） | ⭐ 最好 |
| `const` | 常量 | 主键或唯一索引的**等值**查询，最多返回 1 行 | ⭐ 极好 |
| `eq_ref` | 唯一索引等值 | 每次 JOIN 都用主键/唯一索引定位，**每次最多 1 行** | ⭐ 极好 |
| `ref` | 非唯一索引等值 | 用普通索引等值查询，可能多行 | ✅ 好 |
| `fulltext` | 全文索引 | | ✅ |
| `ref_or_null` | ref + NULL | | ✅ |
| `index` | **全索引扫描** | ⚠️ 扫描整棵索引（比全表略好） | ⚠️ 警惕 |
| `range` | 范围扫描 | 索引范围查询（`>`, `BETWEEN`, `IN`） | ✅ 可接受 |
| `ALL` | **全表扫描** | ⚠️ ⚠️ 没有可用索引 | ❌ **大表要修** |

### 判断标准

| 表行数 | 可接受的 `type` |
|--------|-----------------|
| < 1000（⚠️ 小表） | ⭐ **`ALL` 完全正常**，全表扫描比走索引还快 |
| 1 万 ~ 10 万 | 不该出现 `ALL` |
| > 10 万 | ⚠️ `ALL` = **严重问题** |

⚠️ **⚠️ 「`type=ALL` 就是慢」是错误认知。** 优化器会根据 `rows` 判断——扫描 500 行的成本可能低于走索引（索引也要读根节点+页节点）。

### ⚠️ 明明有索引却不用

```sql
-- 建了索引，但 EXPLAIN 显示 key=NULL
```

**七个排查方向**：

| 原因 | 检查方法 |
|------|---------|
| **⚠️ 索引列上用了函数/运算** | 看 WHERE 条件里有没有 `DATE(col)`、`col*2` |
| **⚠️ 隐式类型转换** | ⚠️ 索引列是 `VARCHAR` 但查询传了数字，或 `INT` 列传了字符串 |
| **⚠️ 前导是 `%` 的 LIKE** | `LIKE '%x'` |
| **⚠️ OR 连接的条件没索引** | `WHERE a=1 OR b=2`，b 没索引则整个失效 |
| ⚠️ 优化器估错行数 | `rows` 与真实值差很多，见 [[mysql/02-中段阶段-专题精讲/索引与查询优化/索引诊断与慢查询优化]] |
| ⚠️ 统计信息过期 | `ANALYZE TABLE` |
| ⚠️ 索引选择性太差 | ⚠️ 索引了 99% 重复的值，优化器判定不如全表扫 |

⭐ **⚠️ 最隐蔽的是隐式类型转换**：

```sql
-- 索引列是 VARCHAR
WHERE phone = 13800138000          -- ❌ ⚠️ 数字和字符串比较，索引失效
WHERE phone = '13800138000'        -- ✅ 加引号

-- 索引列是 INT
WHERE id = '123'                   -- ⚠️ 字符串转数字，索引失效
WHERE id = 123                     -- ✅
```

---

## 三、`key_len`：判断联合索引用到多少

⚠️ **`key_len` = 索引里实际用了多少字节**，可以反推用了联合索引的前几列。

### 计算规则

| 类型 | 每列字节数 |
|------|-----------|
| `INT` | 4 |
| `BIGINT` | 8 |
| `VARCHAR(n)` utf8mb4 | `n × 4 + 2`（+2 是长度前缀） |
| `DATE` / `DATETIME` | 5 / 5~8 |
| `TINYINT` | 1 |

### 读法

```sql
KEY idx_dept_status_hired (dept_id, status, hired_at)   -- TINYINT + TINYINT + DATE
-- key_len = 1  → 只用了 dept_id
-- key_len = 2  → 用了 dept_id + status
-- key_len = 7  → 三个全用了
```

⚠️ **`key_len` 短说明索引没用满**，通常是范围查询导致的。

---

## 四、`rows`：估算的扫描行数

⚠️ **`rows` 是「优化器估算要读多少行」，不是实际值。**

| rows 值 | 含义 |
|--------|------|
| 小 | ✅ 好 |
| 大 | ⚠️ 扫描多行，即使有索引也可能慢 |
| ⚠️ **与真实值差很多** | ⭐ **优化器估错了**，这是选错执行计划的根因 |

### 对比估算与真实

```sql
-- 估算值
EXPLAIN SELECT * FROM orders WHERE user_id = 5;
-- rows: 50000

-- 真实值
SELECT COUNT(*) FROM orders WHERE user_id = 5;
-- 真实: 12
```

⚠️ **估算 50000 实际 12**：优化器以为这个用户有 5 万个订单，选了全表扫描；实际只有 12 行，走索引更快。**这就是「数据倾斜导致的选错索引」。**

📌 解决方法见 [[mysql/02-中段阶段-专题精讲/索引与查询优化/索引诊断与慢查询优化]]。

### `filtered`

⚠️ **`filtered` = `rows` 中有多少比例能通过 WHERE 条件的百分比。**

```
rows = 50000, filtered = 2%
→ 预计 1000 行会真正返回
```

---

## 五、`Extra` 字段：⭐ 最关键的诊断信息

⚠️ **`Extra` 里的每个关键字都在告诉你「数据库做了额外的什么工作」。**

### 好消息（不需要处理）

| 关键字 | 含义 |
|--------|------|
| `Using index` | ⭐ **覆盖索引**（不用回表），性能很好 |
| `Using index condition` | ⭐ **索引下推（ICP）**，8.0 的优化 |
| `Using where` | 常见，一般正常 |
| `Using join buffer (hash join)` | 8.0.18+ 的 hash join，用于无索引的 JOIN |
| `Using temporary` | ⚠️ 用临时表，见下文 |

### 坏消息（需要优化）

#### `Using filesort` ⚠️

⚠️ **`filesort` 不表示在磁盘排序**，只表示「**不能用索引顺序，需要额外排序**」。

```sql
EXPLAIN SELECT * FROM employees ORDER BY age\G
-- Extra: Using filesort
```

⚠️ **判断是否真的慢**：看 [[mysql/02-中段阶段-专题精讲/工具与可观测性/工具链总览]] 里的 `Sort_merge_passes`。⚠️ **⚠️ `Using temporary` + `Using filesort` 同时出现才是危险信号。**

**优化**：给排序列建索引。

#### `Using temporary` ⚠️⚠️

⚠️ **⚠️ 这是最需要警惕的 Extra 值之一**——需要建临时表（内存或磁盘），比 filesort 更贵。

**常见触发场景**：

```sql
-- ① DISTINCT + ORDER BY
SELECT DISTINCT dept_id FROM employees ORDER BY name;

-- ② GROUP BY 和 ORDER BY 是不同的列
SELECT dept_id, COUNT(*) FROM employees GROUP BY dept_id ORDER BY name;
-- ⚠️ 无法用同一个索引既分组又排序

-- ③ UNION（去重）
SELECT a FROM t1 UNION SELECT a FROM t2;
```

**优化方向**：

```sql
-- ② 的优化：让 GROUP BY 和 ORDER BY 用同一个索引
SELECT dept_id, COUNT(*) FROM employees GROUP BY dept_id ORDER BY dept_id;
-- ⚠️ 虽然写法冗余（ORDER BY 排的是分组列），但能用索引避免 temporary
```

⚠️ **⚠️ `Using temporary` 在低数据量时无所谓（临时表在内存），⚠️ 但在大表上会落磁盘，非常慢。**

#### `Impossible WHERE`

```sql
EXPLAIN SELECT * FROM employees WHERE 1=1 AND 2=2;
-- Extra: Impossible WHERE noticed after reading const tables
-- rows: 0
```

⚠️ **优化器发现条件恒为假，直接返回空集**。⚠️ **如果你以为这条查询会返回数据，说明条件写错了**（比如传了个永远不匹配的值）。

#### `Using join buffer`

```sql
-- 关联列没有索引时
Extra: Using join buffer (hash join)
```

⚠️ **MySQL 8.0.18+ 新增了 hash join**，⚠️ 无索引的大表 JOIN 用它比 Nested Loop 快很多。但**根本解法还是给关联列建索引**。

#### `Select tables optimized away`

⚠️ **优化器发现这条 SELECT 里没有需要查的表**（比如只查常量或已经在索引里的最大值）。

```sql
EXPLAIN SELECT MAX(id) FROM employees WHERE dept_id = 1;
-- Extra: Select tables optimized away
-- ⚠️ 极快，直接从索引末尾取值
```

---

## 六、完整的优化流程

⭐ **这个流程应该成为肌肉记忆。**

```
① 找到慢查询（慢日志 / performance_schema / APM）
        ↓
② EXPLAIN 看执行计划
        ↓
③ 看三件事：
   ├── type 是什么？（ALL/index = 危险）
   ├── key 是什么？（NULL = 索引没用上）
   └── Extra 有什么？（filesort / temporary = 危险）
        ↓
④ 定位问题类型
   ├── key=NULL → 索引没用上（看 type 和 WHERE 条件）
   ├── type=ALL → 没有索引 / 索引没价值 / 优化器估错
   ├── rows 太大 → 索引选择性差 / 缺索引
   ├── Extra 有 filesort → 排序列没索引
   ├── Extra 有 temporary → 查询写法问题
   └── rows 与真实差很大 → 优化器估错（统计信息/直方图）
        ↓
⑤ EXPLAIN ANALYZE 对比真实耗时（⚠️ 别跑写语句）
        ↓
⑥ 优化：
   ├── 加索引 / 改复合索引顺序
   ├── 改写查询（避免 temporary）
   ├── ANALYZE TABLE 更新统计信息
   └── 加直方图（数据倾斜时）
        ↓
⑦ 重新 EXPLAIN 验证
        ↓
⑧ 优化索引下推
```

---

## 七、四个实战案例

### 案例 1：明明有索引却全表扫描

```sql
-- 表：users，索引 idx_phone (phone VARCHAR(20))
EXPLAIN SELECT * FROM users WHERE phone = 13800138000;
-- type: ALL, key: NULL   ← ⚠️
```

**原因**：`phone` 是 VARCHAR，查询传的是数字 → **隐式类型转换** → 索引失效。

```sql
EXPLAIN SELECT * FROM users WHERE phone = '13800138000';
-- type: ref, key: idx_phone   ← ✅
```

⭐ **📌 记忆锚点：索引列的类型要和查询值的类型严格一致。**

### 案例 2：范围查询后联合索引断了

```sql
KEY idx_created_status (created_at, status)
WHERE created_at >= '2026-01-01' AND status = 'paid'
-- key_len: 5（只有 created_at），status 用不上
```

**影响**：`status` 只能在回表后过滤。⚠️ 但 **ICP（索引下推）能减少回表次数**，所以不一定慢。

### 案例 3：`Using temporary` + `Using filesort`

```sql
-- 慢查询
SELECT dept_id, COUNT(*) FROM employees
GROUP BY dept_id
ORDER BY name;
-- Extra: Using temporary; Using filesort   ← ⚠️ 双坏
```

**优化**：

```sql
-- 让 GROUP BY 和 ORDER BY 用同一个索引
KEY idx_dept (dept_id)
SELECT dept_id, COUNT(*) FROM employees
GROUP BY dept_id
ORDER BY dept_id;      -- ⚠️ 写法冗余但能用索引
-- Extra: Using index    ← ✅
```

### 案例 4：排序导致回表暴增

```sql
-- 索引 (dept_id, salary)，但要展示所有列
SELECT * FROM employees WHERE dept_id = 1 ORDER BY salary LIMIT 20;
-- Extra: Using index condition
-- ⚠️ 需要回表 20 次拿所有列
```

**优化**：**覆盖索引**——把要展示的列也放进索引：

```sql
KEY idx_dept_salary_full (dept_id, salary, name, email, created_at)
-- Extra: Using index   ← ⭐ 不回表了
```

⚠️ **代价**：索引变大，写入变慢，⚠️ 且**覆盖索引不能用于排序以外的场景**。这是权衡。

📌 详见 [[mysql/02-中段阶段-专题精讲/索引与查询优化/覆盖索引与回表优化]]。

---

## 八、排查

```sql
-- 逐行看得更清楚（⚠️ 加 \G）
EXPLAIN SELECT ...\G

-- 树状（8.0.18+）
EXPLAIN FORMAT=TREE SELECT ...;

-- 真实耗时（⚠️ 会执行！写语句危险）
EXPLAIN ANALYZE SELECT ...;

-- ⚠️ 写语句先转成 SELECT 再 ANALYZE
EXPLAIN ANALYZE SELECT ... FROM (SELECT ...) t;

-- 优化器的完整决策过程
SET optimizer_trace = 'enabled=on';
SELECT ...;
SELECT * FROM information_schema.OPTIMIZER_TRACE\G
SET optimizer_trace = 'enabled=off';
```

📌 optimizer_trace 的读法见 [[mysql/02-中段阶段-专题精讲/索引与查询优化/索引诊断与慢查询优化]]。

---

## 九、陷阱

1. **❌ 不看 EXPLAIN 就优化。** ⚠️ 凭直觉改索引是浪费时间。
2. **❌ 看到 `type=ALL` 就改。** ⚠️ 小表上这是正常的。
3. **❌ 以为 `Using filesort` 是在磁盘排序。** 它只表示「不能用索引顺序」。
4. **❌ 忽略 `Using temporary`。** ⚠️ 比 filesort 更贵。
5. **❌ 对写语句跑 `EXPLAIN ANALYZE`。** ⚠️ **会真的改数据**。
6. **❌ 只看 `key` 不看 `key_len`。** 联合索引用了一半也算没用满。
7. **❌ 忽略隐式类型转换。** ⭐ 最隐蔽的索引失效原因。
8. **❌ 不对比 `rows` 和真实行数。** 优化器估错是选错计划的根因。

---

## 小段

- **优化第一步永远是 `EXPLAIN`**
- `type` 优劣：`const` > `eq_ref` > `ref` > `range` > `index` > `ALL`。⚠️ **小表上 `ALL` 正常**
- `key=NULL` = 索引没用上。**七个原因**：函数/运算、**隐式类型转换**、`%` 开头的 LIKE、OR 的某列无索引、统计估算错、统计信息过期、选择性太差
- `key_len` 能反推联合索引用了几列；⚠️ **范围查询之后的部分用不上**
- ⚠️ **`rows` 是估算值**，与真实值差很多说明优化器估错了
- ⭐ **`Extra` 是最有诊断价值的信息**：`Using index`（好）/ `Using index condition`（好）/ ⚠️ `Using filesort`（不能索引排序）/ ⚠️⚠️ `Using temporary`（最危险）
- **`EXPLAIN ANALYZE`（8.0.18+）给真实耗时**，⚠️ **会真的执行，写语句绝对不能跑**
- 完整流程：慢日志 → EXPLAIN → 看 type/key/Extra → ANALYZE 对比 → 优化 → 验证

## 练习

1. `type` 从好到坏排一下顺序。什么情况下 `ALL` 是正常的？
2. 建了索引但 `key` 显示 NULL，列出你的排查方向（至少五个）。
3. 为什么 `WHERE phone = 13800138000` 可能让索引失效？怎么判断？
4. `Using filesort` 真的在磁盘排序吗？什么情况下它才值得担心？
5. 什么查询会产生 `Using temporary`？给两个例子和优化方向。
6. 为什么 `EXPLAIN ANALYZE` 对写语句很危险？
7. 怎么判断「优化器估错了行数」？这个问题的后果是什么？

## 导航

[[mysql/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 上一篇：[[mysql/02-中段阶段-专题精讲/索引与查询优化/索引类型与选型]] · 下一篇：[[mysql/02-中段阶段-专题精讲/索引与查询优化/覆盖索引与回表优化]]
