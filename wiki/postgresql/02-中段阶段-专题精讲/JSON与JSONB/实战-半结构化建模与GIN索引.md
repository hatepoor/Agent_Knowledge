---
tags: [concept, PostgreSQL, 中段, JSONB, 建模, GIN, 实战]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# 实战:半结构化建模与 GIN 索引

## 场景

电商平台的商品表。问题:不同品类的商品属性差异很大。

- 手机:屏幕尺寸、电池容量、CPU
- 服装:尺码表、面料、季节
- 图书:ISBN、页数、出版社

**方案对比**

| 方案 | 表结构 | 问题 |
|------|--------|------|
| 宽表 + 大量可空列 | 所有品类属性都建列 | 列数爆炸,大量 NULL,加属性要 DDL |
| EAV(属性-值表) | `(product_id, attr_name, attr_value)` | JOIN 地狱,类型全是 text |
| **核心列 + JSONB 属性** | 常查字段升列,差异属性进 JSONB | **推荐** |
| 每品类一张表 | 独立表 | 表数量爆炸,跨品类查询难 |

## 一、表设计

```sql
CREATE TABLE products (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku         text NOT NULL,
    category    text NOT NULL,          -- 常查、需约束 → 列
    name        text NOT NULL,
    price       numeric(12,2) NOT NULL, -- 常查、需精确 → 列
    stock       integer NOT NULL DEFAULT 0,
    attrs       jsonb NOT NULL DEFAULT '{}'::jsonb,   -- 品类差异属性
    created_at  timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT products_sku_uk UNIQUE (sku),
    CONSTRAINT products_price_ck CHECK (price >= 0),
    CONSTRAINT products_attrs_obj_ck CHECK (jsonb_typeof(attrs) = 'object')
);

-- 高频检索的 JSONB 键升为生成列 + 索引(推荐做法!)
ALTER TABLE products
  ADD COLUMN brand text GENERATED ALWAYS AS (attrs->>'brand') STORED;

CREATE INDEX products_brand_idx ON products (brand);
```

> **"生成列"是关键技巧**:把"JSONB 里高频查询的键"提取成 `STORED` 生成列,就能用普通 B-tree 索引,避免 GIN 索引的体积与统计信息问题。记住 PG18 起生成列默认是 `VIRTUAL`,**要给生成列建索引必须显式写 `STORED`**。

数据示例:

```sql
INSERT INTO products (sku, category, name, price, attrs) VALUES
('P-1001','phone','旗舰手机',5999,'{"brand":"X","screen":6.7,"battery":5000,"tags":["5g","flagship"]}'),
('P-1002','phone','入门手机', 999,'{"brand":"Y","screen":6.1,"battery":4000,"tags":["4g"]}'),
('P-2001','clothes','纯棉T恤',  99,'{"brand":"Z","size":["S","M","L"],"season":"summer"}'),
('P-2002','clothes','羽绒服',  699,'{"brand":"Z","size":["M","L"],"season":"winter"}'),
('P-3001','book','数据库原理', 128,'{"publisher":"某社","pages":560,"lang":"zh"}');
```

## 二、查询模式与索引匹配

### 模式 1:精确匹配某个键值(高频)

```sql
-- ✅ 方案 A:生成列 + B-tree(推荐)
CREATE INDEX products_brand_idx ON products (brand);
SELECT * FROM products WHERE brand = 'Z';

-- 方案 B:表达式索引
CREATE INDEX products_brand_expr_idx ON products ((attrs->>'brand'));
SELECT * FROM products WHERE attrs->>'brand' = 'Z';

-- 方案 C:GIN 包含
SELECT * FROM products WHERE attrs @> '{"brand":"Z"}';
```

**选择建议**

| 键的查询频率 | 推荐 |
|--------------|------|
| 很高(主过滤条件) | 生成列 / 表达式索引 + B-tree |
| 中等(辅助条件) | GIN `jsonb_ops` |
| 低 | 不建索引,靠其他条件先过滤 |

### 模式 2:多键组合过滤

```sql
-- 数值范围 + 键存在性:表达式索引 + 组合条件
CREATE INDEX products_screen_idx ON products (((attrs->>'screen')::numeric));

SELECT * FROM products
WHERE (attrs->>'screen')::numeric >= 6.5
  AND attrs @> '{"tags":["5g"]}';
```

### 模式 3:数组包含

```sql
-- attrs->tags 是数组,判断包含某标签
SELECT * FROM products WHERE attrs->'tags' @> '["5g"]'::jsonb;
SELECT * FROM products WHERE attrs @> '{"tags":["5g"]}';

-- 判断数组中任一元素匹配
SELECT * FROM products WHERE attrs->'tags' ?| array['5g','flagship'];

-- 标签必须全部存在
SELECT * FROM products WHERE attrs->'tags' ?& array['5g','flagship'];
```

索引:

```sql
CREATE INDEX products_attrs_gin ON products USING gin (attrs);                  -- 支持 ? ?| ?& @>
CREATE INDEX products_tags_gin  ON products USING gin ((attrs->'tags'));        -- 只针对 tags
```

### 模式 4:键存在性

```sql
SELECT * FROM products WHERE attrs ? 'battery';        -- 有 battery 键
SELECT * FROM products WHERE NOT attrs ? 'season';
```

> 必须用 `jsonb_ops`(默认 GIN)。`jsonb_path_ops` **不支持** `?`。

### 模式 5:嵌套路径查询

```sql
INSERT INTO products (sku, category, name, price, attrs) VALUES
('P-1003','phone','折叠屏',8999,
 '{"brand":"X","spec":{"display":{"size":7.6,"refresh":120},"battery":4800}}');

-- SQL/JSON 路径
SELECT * FROM products
WHERE jsonb_path_exists(attrs, '$.spec.display.refresh ? (@ >= 120)');

-- 表达式索引支持嵌套
CREATE INDEX products_refresh_idx ON products (((attrs#>'{spec,display,refresh}')::int));
SELECT * FROM products WHERE (attrs#>'{spec,display,refresh}')::int >= 120;
```

### 模式 6:JSONB 内的唯一业务键

```sql
-- 外部系统 ID 存在 JSONB 里,但仍要保证唯一
CREATE UNIQUE INDEX products_ext_id_uk ON products ((attrs->>'ext_id'))
    WHERE attrs ? 'ext_id';    -- 部分索引,只对含该键的行生效
```

## 三、GIN 索引的调优

### 先理解 pending list 是什么

GIN 索引的写入代价高(每个被索引的元素都要新增一条索引项)。为了不让写入被拖慢,PostgreSQL 提供了 **fastupdate** 机制:

```
新数据写入
   │
   ▼
先写进 pending list(一个单纯的链式列表,写入极快)
   │  累积到阈值(gin_pending_list_limit)或收到清理请求时
   ▼
批量合并进 GIN 主索引结构(entry tree / posting list)
```

**代价转移**:写入变快了,但代价转移到读取——每次查询都要额外扫一遍 pending list。

| 配置 | 写入速度 | 查询速度 | pending list |
|------|----------|----------|--------------|
| `fastupdate = on`(默认) | 快 | 稍慢(需扫 pending list) | 会累积,需定期合并 |
| `fastupdate = off` | 慢 | 快(直接查主结构) | 不存在 |

```sql
-- 建索引时显式指定(GIN 索引默认就是 fastupdate = on)
CREATE INDEX products_attrs_gin ON products USING gin (attrs)
    WITH (fastupdate = on, gin_pending_list_limit = 4MB);
--        ↑ 开启延迟合并          ↑ pending list 达到 4MB 就自动触发合并

-- 查看索引体积(体积会随 pending list 累积而增长)
SELECT pg_size_pretty(pg_relation_size('products_attrs_gin')) AS index_size;

-- gin_clean_pending_list(索引名):手动把 pending list 合并进主索引结构
-- 返回值 = 本次合并掉的页数;没有 pending 项时返回 0
SELECT gin_clean_pending_list('products_attrs_gin'::regclass);

-- 排查 GIN 索引是否真的被用上
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM products WHERE attrs @> '{"brand":"Z"}';
-- 期望看到 "Bitmap Index Scan on products_attrs_gin"
-- 若出现 Seq Scan,说明索引未被使用(常见原因:未 ANALYZE,或统计信息过期)
```

> **`'索引名'::regclass` 是什么**:`::regclass` 把"索引名的字符串"转换成"索引的对象标识"(OID)。`gin_clean_pending_list` 需要的是对象标识而不是名字文本,所以要转换。多数系统函数两种写法都能接受,但带 `regclass` 参数的更严格,名字写错会直接报错而不是静默返回空。
>
> **什么时候要手动调用 `gin_clean_pending_list`**:仅批量写入之后需要"立刻恢复查询性能"时才用。日常无需手工干预——写入量达到 `gin_pending_list_limit` 会自动合并,查询也会触发。

**大批量导入的推荐流程**

```sql
-- 1) 删索引(或先建无索引表)
DROP INDEX products_attrs_gin;
-- 2) COPY 导入
\copy products FROM 'products.csv' WITH (FORMAT csv, HEADER)
-- 3) 重建索引(比边导入边维护快得多)
CREATE INDEX products_attrs_gin ON products USING gin (attrs);
-- 4) 更新统计信息
ANALYZE products;
```

## 四、从 JSONB 生成关系视图

JSONB 适合存储,但报表通常需要关系结构。用视图把 JSONB"投影"成列:

```sql
CREATE VIEW v_products AS
SELECT
    id, sku, category, name, price, stock,
    attrs->>'brand'              AS brand,
    (attrs->>'screen')::numeric  AS screen_size,
    (attrs->>'battery')::int     AS battery,
    (attrs#>'{spec,display,refresh}')::int AS refresh_rate,
    attrs->'tags'                AS tags,
    attrs                        AS attrs_raw
FROM products;

-- 查询视图就像查普通表
SELECT category, brand, avg(screen_size) FROM v_products GROUP BY 1,2;
```

> **注意**:视图中的表达式**不能自动享受索引**。若某列是报表高频入口,应把它升级为**物化视图**或**生成列**。

```sql
-- 物化视图:适合"读多、可容忍延迟"的报表
CREATE MATERIALIZED VIEW mv_product_stats AS
SELECT category, brand, count(*) AS cnt, avg(price) AS avg_price
FROM v_products GROUP BY 1,2;

CREATE UNIQUE INDEX ON mv_product_stats (category, brand);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_stats;   -- 不阻塞读(需唯一索引)
```

## 五、JSONB 转关系表(反向投影)

从事件流 JSONB 抽取到宽表:

```sql
CREATE TABLE events_flat (
    id       bigint,
    event_at timestamptz,
    user_id  bigint,
    city     text,
    amount   numeric
);

INSERT INTO events_flat (id, event_at, user_id, city, amount)
SELECT e.id,
       e.created_at,
       (e.payload->>'user_id')::bigint,
       e.payload->>'city',
       (e.payload->>'amount')::numeric
FROM events e
WHERE e.payload @> '{"type":"order"}';       -- 只抽特定类型,配合 GIN 索引
```

**用 `jsonb_to_record` 减少样板**:

```sql
INSERT INTO events_flat (id, user_id, city, amount)
SELECT e.id, r.user_id, r.city, r.amount
FROM events e,
     jsonb_to_record(e.payload) AS r(user_id bigint, city text, amount numeric)
WHERE (e.payload->>'type') = 'order';
```

## 六、常见性能反模式

| 反模式 | 后果 | 修正 |
|--------|------|------|
| 所有字段放一个 JSONB | 无法约束、无法统计、索引巨大 | 核心字段升列 |
| 频繁更新 JSONB 单个键 | 每次重写整行 + 全部索引 | 把该键升为独立列 |
| 只用 GIN,不用表达式索引 | 简单等值查询要走 Bitmap 合并,慢 | 高频键建 B-tree 表达式索引 |
| JSONB 值过大(> 数十 KB) | 进 TOAST,读取放大 | 拆分或改为子表 |
| 混用 `@>` 与 `?` 但索引是 `jsonb_path_ops` | `?` 走不了索引 | 改回 `jsonb_ops` |
| 忘了 `ANALYZE` | 规划器误估行数,选错计划 | 导入后 `ANALYZE` |

## 七、验证清单

```sql
-- 1) 索引是否被用
EXPLAIN (ANALYZE, BUFFERS) SELECT ... WHERE brand = 'Z';

-- 2) 索引体积是否合理
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes WHERE relname = 'products';

-- 3) 索引是否真被访问
SELECT indexrelname, idx_scan FROM pg_stat_user_indexes WHERE relname = 'products';
-- idx_scan = 0 且持续为 0 → 可能是无用索引

-- 4) 统计信息是否到位
SELECT * FROM pg_stats WHERE tablename = 'products' AND attname = 'attrs';
```

## 相关页面

- [[postgresql/02-中段阶段-专题精讲/JSON与JSONB/专题概览]]
- [[postgresql/02-中段阶段-专题精讲/索引实战应用/索引类型选型与设计]]
- [[postgresql/02-中段阶段-专题精讲/视图函数与触发器/专题概览]]
