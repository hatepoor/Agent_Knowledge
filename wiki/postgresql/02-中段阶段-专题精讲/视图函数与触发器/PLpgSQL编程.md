---
tags: [concept, PostgreSQL, 中段, PLpgSQL, 存储过程, 异常处理, 游标]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# PL/pgSQL 编程

## 一、基本结构

```sql
CREATE FUNCTION add_user(p_email text, p_name text)
RETURNS bigint
LANGUAGE plpgsql
AS $$
DECLARE
    v_id bigint;                      -- 局部变量
BEGIN
    INSERT INTO users (email, name) VALUES (p_email, p_name)
    RETURNING id INTO v_id;
    RETURN v_id;
EXCEPTION
    WHEN unique_violation THEN        -- 异常处理
        SELECT id INTO v_id FROM users WHERE email = p_email;
        RETURN v_id;
END;
$$;
```

**四段式**:`DECLARE`(可选)→ `BEGIN` → 主体 → `EXCEPTION`(可选)→ `END`。

> **注意**:可用 `$$` 或 `$body$` 等作为美元引号;复合类型与变量名易与列名冲突,建议变量统一加 `v_` / `p_` 前缀。

## 二、变量与赋值

```sql
DECLARE
    v_count    integer := 0;
    v_sum      numeric(12,2) DEFAULT 0;
    v_name     text;
    v_row      orders%ROWTYPE;        -- 行类型:跟表结构同步
    v_amount   orders.total%TYPE;     -- 列类型:跟列类型同步
BEGIN
    v_count := v_count + 1;
    v_sum   := v_sum + 100;

    SELECT name INTO v_name FROM users WHERE id = 1;
    SELECT * INTO v_row  FROM orders WHERE id = 1;

    v_row.status := 'done';           -- 访问行类型字段
END;
```

**`%TYPE` / `%ROWTYPE` 的价值**:表结构变化时函数无需修改,类型自动跟随。**强烈推荐使用。**

**块级嵌套与作用域**

```sql
BEGIN
    DECLARE v_inner int := 1;         -- 内层块
    BEGIN
        RAISE NOTICE 'inner=%', v_inner;
    END;
END;
```

内层块可访问外层变量;同名变量遮蔽外层。

## 三、流程控制

```sql
-- IF
IF v_n > 10 THEN
    ...
ELSIF v_n > 5 THEN
    ...
ELSE
    ...
END IF;

-- CASE
CASE v_status
    WHEN 'paid' THEN v_label := '已付';
    WHEN 'done' THEN v_label := '完成';
    ELSE v_label := '其他';
END CASE;

-- 简单循环
LOOP
    EXIT WHEN v_i > 10;
    v_i := v_i + 1;
END LOOP;

-- WHILE
WHILE v_i <= 10 LOOP
    v_i := v_i + 1;
END LOOP;

-- FOR(数值区间)
FOR i IN 1..10 LOOP
    RAISE NOTICE '%', i;
END LOOP;
FOR i IN REVERSE 10..1 LOOP ... END LOOP;      -- 逆序
FOR i IN 1..10 BY 2 LOOP ... END LOOP;         -- 步长

-- FOR(查询结果集)★最常用
FOR v_rec IN SELECT id, total FROM orders WHERE status = 'pending' LOOP
    UPDATE orders SET status = 'processed' WHERE id = v_rec.id;
END LOOP;

-- FOREACH(遍历数组)
FOREACH v_tag IN ARRAY v_tags LOOP
    RAISE NOTICE '%', v_tag;
END LOOP;

-- CONTINUE
FOR i IN 1..10 LOOP
    CONTINUE WHEN i % 2 = 0;
    RAISE NOTICE '%', i;
END LOOP;
```

**`RETURN NEXT` / `RETURN QUERY`:返回集合**

```sql
CREATE FUNCTION orders_of_user(p_user bigint)
RETURNS SETOF orders
LANGUAGE plpgsql STABLE AS $$
BEGIN
    RETURN QUERY SELECT * FROM orders WHERE user_id = p_user;
END;
$$;

-- 或逐行构造
CREATE FUNCTION gen_nums(n int) RETURNS SETOF int
LANGUAGE plpgsql AS $$
BEGIN
    FOR i IN 1..n LOOP
        RETURN NEXT i;
    END LOOP;
    RETURN;
END;
$$;

SELECT * FROM gen_nums(5);
```

> **`RETURN QUERY` 通常比 `RETURN NEXT` 逐行更快**,因为它可批量处理。

## 四、`RAISE` 消息与异常

```sql
RAISE NOTICE 'value=%', v_val;          -- 提示级
RAISE WARNING 'careful';
RAISE EXCEPTION 'invalid amount: %', v_amt;                 -- 抛错
RAISE EXCEPTION USING ERRCODE = '22000', MESSAGE = 'bad';   -- 指定 SQLSTATE
RAISE DEBUG 'only when debug enabled';
```

**级别**:`DEBUG` < `LOG` < `INFO` < `NOTICE` < `WARNING` < `EXCEPTION`。`client_min_messages` 控制哪些会被发给客户端。

**异常捕获**

```sql
BEGIN
    INSERT INTO t VALUES (1);
EXCEPTION
    WHEN unique_violation THEN
        RAISE NOTICE 'duplicate, skipping';
    WHEN foreign_key_violation THEN
        RAISE NOTICE 'missing parent';
    WHEN division_by_zero THEN
        RAISE NOTICE 'div by zero';
    WHEN OTHERS THEN                     -- 兜底
        RAISE WARNING 'unexpected: % (%)', SQLERRM, SQLSTATE;
        -- 若要重新抛出:
        -- RAISE;
END;
```

**`GET STACKED DIAGNOSTICS` 获取详情**

```sql
EXCEPTION WHEN OTHERS THEN
    GET STACKED DIAGNOSTICS
        v_msg  = MESSAGE_TEXT,
        v_state = RETURNED_SQLSTATE,
        v_col  = COLUMN_NAME,
        v_tbl  = TABLE_NAME;
    RAISE EXCEPTION 'err % in %.%: %', v_state, v_tbl, v_col, v_msg;
```

> **`WHEN OTHERS` 的代价**:它会**吞掉**未预期的错误,且**无法提交部分工作**——异常块内已执行的 DML 会被回滚到块的保存点。用于"确实知道要忽略什么"时才写,并在其中重新抛出未知错误。

## 五、`EXCEPTION` 与子事务

**重要机制**:PL/pgSQL 的 `EXCEPTION` 块会在入口自动建立一个**子事务(SAVEPOINT)**。

| 后果 | 说明 |
|------|------|
| 块内 DML 出错可回滚,块外不受影响 | 这正是 `EXCEPTION` 的价值 |
| 每次进入 `EXCEPTION` 块消耗一个 xid | 循环中大量使用会**快速消耗 xid** |
| 需要 `max_locks_per_transaction` 等的连带消耗 | 子事务持有锁 |

```sql
-- ⚠️ 反模式:循环内用 EXCEPTION 做逐行容错(10 万次 = 10 万个子事务)
FOR v_rec IN SELECT * FROM staging LOOP
    BEGIN
        INSERT INTO target VALUES (v_rec.*);
    EXCEPTION WHEN unique_violation THEN NULL;
    END;
END LOOP;
```

**推荐替代**:

```sql
-- ✅ 用 ON CONFLICT 代替异常捕获
INSERT INTO target SELECT * FROM staging
ON CONFLICT DO NOTHING;

-- ✅ 或者先过滤重复
INSERT INTO target
SELECT s.* FROM staging s
WHERE NOT EXISTS (SELECT 1 FROM target t WHERE t.id = s.id);
```

**监控子事务消耗**:

```sql
SHOW max_locks_per_transaction;   -- PG19 默认由 64 改为 128
-- 观察事务号消耗速度
SELECT txid_current_if_assigned();
```

## 六、游标

**显式游标**(适合需要分段处理的大结果集)

```sql
CREATE FUNCTION process_orders() RETURNS void
LANGUAGE plpgsql AS $$
DECLARE
    v_cur CURSOR FOR SELECT id, total FROM orders WHERE status = 'pending';
    v_id bigint;
    v_total numeric;
BEGIN
    OPEN v_cur;
    LOOP
        FETCH v_cur INTO v_id, v_total;
        EXIT WHEN NOT FOUND;
        -- 处理
    END LOOP;
    CLOSE v_cur;
END;
$$;
```

**带参数的游标**

```sql
v_cur CURSOR (p_status text) FOR SELECT * FROM orders WHERE status = p_status;
OPEN v_cur('paid');
```

**游标选项**

```sql
v_cur CURSOR FOR SELECT ... FOR UPDATE;             -- 行锁
v_cur NO SCROLL CURSOR FOR SELECT ... ;             -- 只能前进,可能更快
```

> **`FOR` 循环优于显式游标**:`FOR rec IN SELECT ... LOOP` 内部就用隐式游标,更简洁,且不会有忘记 `CLOSE` 的问题。

**大结果集的性能陷阱**

```sql
-- ⚠️ PL/pgSQL 的 FOR 循环会一次性物化整个结果集(除非用游标并按需 FETCH)
FOR rec IN SELECT * FROM huge_table LOOP ... END LOOP;

-- ✅ 若结果集极大,分批处理
LOOP
    WITH batch AS (
        SELECT id FROM huge_table WHERE processed = false ORDER BY id LIMIT 10000
    )
    UPDATE huge_table h SET processed = true FROM batch b WHERE h.id = b.id;
    EXIT WHEN NOT FOUND;
    COMMIT;                          -- 需在 PROCEDURE 中,或外层控制
END LOOP;
```

## 七、性能要点

| 要点 | 说明 |
|------|------|
| **不要逐行 DML** | `FOR ... UPDATE` 逐行远慢于集合操作 |
| 优先集合式 SQL | 能用一条 `UPDATE ... FROM` 就别用循环 |
| 用 `sql` 语言函数 | 可被内联优化 |
| 声明正确的易变性 | `STABLE` / `IMMUTABLE` 能显著改善计划 |
| 缓存查询计划 | PL/pgSQL 会缓存函数内 SQL 的计划(注意通用计划问题) |
| 少用 `EXCEPTION` 在循环内 | 每个异常块一个子事务 |
| `SET search_path` | 函数内显式设置,避免注入与计划差异 |

```sql
-- ⚠️ 反模式:逐行更新
FOR rec IN SELECT id, amount FROM orders WHERE status = 'paid' LOOP
    UPDATE orders SET total = rec.amount * 1.1 WHERE id = rec.id;
END LOOP;

-- ✅ 集合式
UPDATE orders SET total = amount * 1.1 WHERE status = 'paid';
```

**`PERFORM`**:在 PL/pgSQL 中执行查询但丢弃结果。

```sql
PERFORM 1 FROM orders WHERE id = 1;    -- 相当于 SELECT 但不返回
IF NOT FOUND THEN ... END IF;
```

**`FOUND`**:上一条 `SELECT INTO` / `PERFORM` / `UPDATE` 是否影响行。

## 八、调试技巧

```sql
-- 1) 打印消息
RAISE NOTICE 'v_x=%', v_x;

-- 2) 查看函数定义
\df+ function_name
SELECT pg_get_functiondef('function_name'::regproc);

-- 3) 分段测试:把函数体拆成多条 SQL 手工跑

-- 4) 用 RAISE 计时
DECLARE t0 timestamptz := clock_timestamp();
...
RAISE NOTICE 'elapsed=%', clock_timestamp() - t0;

-- 5) pg_stat_statements 会归类函数内的 SQL(需 track 设置)
SELECT query FROM pg_stat_statements WHERE query ILIKE '%INSERT INTO users%';
```

## 相关页面

- [[postgresql/02-中段阶段-专题精讲/视图函数与触发器/专题概览]]
- [[postgresql/01-入门阶段-基础教程/第06章-事务与权限入门/6.1-事务基础]]
- [[postgresql/03-进阶阶段-深度剖析/事务与MVCC内核/原理剖析-可见性与事务状态]] — 子事务与 xid 消耗
