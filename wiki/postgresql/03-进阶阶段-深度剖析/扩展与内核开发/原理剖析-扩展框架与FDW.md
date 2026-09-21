---
tags: [concept, PostgreSQL, 进阶, 扩展框架, FDW, 钩子, 后台worker]
type: concept
created: 2026-09-19
updated: 2026-09-19
---

# 原理剖析 · 扩展框架与 FDW

## ① 现象

- 装了扩展但函数报 "function does not exist" 或 "not loaded"
- `CREATE EXTENSION` 需要超级用户
- 跨库查询(`postgres_fdw`)把整表拉过来才过滤,极慢
- 扩展在 `pg_upgrade` 后失效
- 有些扩展必须重启,有些不用

## ② 机制

### 扩展的三个组成部分

```
<name>.control         ← 元数据(默认版本、是否需预加载、依赖、是否可信)
<name>--1.0.sql        ← 安装脚本(DDL:CREATE FUNCTION/TYPE/OPERATOR)
<name>.so              ← 共享库(C 函数实现)
```

**`.control` 的关键字段**

| 字段 | 作用 |
|------|------|
| `default_version` | 默认安装版本 |
| `module_pathname` | 共享库路径 |
| `relocatable` | 是否可安装到任意 schema |
| `schema` | 强制安装到的 schema |
| `requires` | 依赖的其他扩展 |
| `trusted` | **是否允许非超级用户安装** |
| `superuser` | 是否必须超级用户创建 |

```bash
# 查看 control 文件位置
ls $(pg_config --sharedir)/extension/*.control
cat $(pg_config --sharedir)/extension/pg_trgm.control
```

```sql
-- 通过 SQL 查看
SELECT name, default_version, installed_version, comment, trusted
FROM pg_available_extensions ORDER BY name;
SELECT * FROM pg_available_extension_versions WHERE name = 'pg_stat_statements';
```

### 为什么有些扩展需要重启

| 类型 | 原因 | 例子 |
|------|------|------|
| 纯 SQL 对象 | 只用 `CREATE FUNCTION ... LANGUAGE sql/plpgsql` | `pg_trgm` 的部分、`hstore` |
| **需要预加载** | C 代码需要注册**钩子(hook)**、**后台 worker**、**共享内存** | `pg_stat_statements`、`pg_cron`、`auto_explain`、`pg_stat_kcache` |
| 需要共享内存 | 需在实例启动时分配 | `pg_stat_statements` |

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements,auto_explain,pg_cron'
```

> **`shared_preload_libraries` 必须在实例启动时加载**,之后 `CREATE EXTENSION` 才能找到已注册的函数。这是"装了却报错"的常见原因。

**查看当前预加载了什么**

```sql
SHOW shared_preload_libraries;
-- 或从 pg_settings
SELECT setting FROM pg_settings WHERE name = 'shared_preload_libraries';
```

### 钩子(Hook)机制

钩子是 PostgreSQL 提供的**扩展点**:扩展可以替换或包裹内核的某个函数。

| 钩子 | 触发点 | 典型用途 |
|------|--------|----------|
| `ProcessUtility_hook` | 执行 DDL/工具命令 | 审计 DDL、拦截 `CREATE TABLE` |
| `ExecutorStart_hook` | 执行器启动 | 强制超时、注入条件 |
| `ExecutorRun_hook` | 执行器运行 | 统计、限制行数 |
| `ExecutorEnd_hook` | 执行器结束 | 资源统计 |
| `planner_hook` | 规划阶段 | 改写计划、加提示 |
| `post_parse_analyze_hook` | 解析后 | 语句改写 |
| `shmem_request_hook` | 共享内存申请 | 分配自定义共享内存(PG15+) |
| `emit_log_hook` | 日志输出 | 日志采集 |
| `object_access_hook` | 对象访问 | 权限审计 |
| `ClientAuthentication_hook` | 认证 | 自定义认证 |

**钩子的链式调用模式**

```c
/* 典型写法:保存上一个钩子,自己的钩子内先/后调用它 */
static ProcessUtility_hook_type prev_ProcessUtility = NULL;

static void my_ProcessUtility(PlannedStmt *pstmt, ...)
{
    /* 前置逻辑 */
    if (prev_ProcessUtility)
        prev_ProcessUtility(pstmt, ...);
    else
        standard_ProcessUtility(pstmt, ...);
    /* 后置逻辑 */
}

/* _PG_init 中安装 */
void _PG_init(void)
{
    prev_ProcessUtility = ProcessUtility_hook;
    ProcessUtility_hook = my_ProcessUtility;
}
```

> **`_PG_init`** 是扩展共享库的入口函数,在库被加载时调用(预加载时在实例启动阶段)。**钩子必须在 `_PG_init` 中安装。**

### 后台 worker(Background Worker)

扩展可请求内核启动自己的后台进程,用于定期任务或外部数据拉取。

```c
/* _PG_init 中注册 */
BackgroundWorker worker;
memset(&worker, 0, sizeof(worker));
worker.bgw_flags = BGWORKER_SHMEM_ACCESS | BGWORKER_BACKEND_DATABASE_CONNECTION;
worker.bgw_start_time = BgWorkerStart_RecoveryFinished;
snprintf(worker.bgw_library_name, BGW_MAXLEN, "my_ext");
snprintf(worker.bgw_function_name, BGW_MAXLEN, "my_worker_main");
worker.bgw_name = "my background worker";
worker.bgw_restart_time = 10;      /* 崩溃后 10 秒重启;0 = 不重启 */
RegisterBackgroundWorker(&worker);
```

| 类型 | 说明 |
|------|------|
| 静态 worker | 在 `_PG_init` 中 `RegisterBackgroundWorker`,随实例启动 |
| 动态 worker | 运行期用 `RegisterDynamicBackgroundWorker` 启动 |

参数:`max_worker_processes`、`max_parallel_workers` 都会影响可用槽位。
`pg_stat_activity.backend_type` 会显示 `background worker`。

### 扩展的安全模型

| 风险点 | 说明 |
|--------|------|
| C 代码可访问内核 | 恶意/有 bug 的扩展可导致任意代码执行或崩溃 |
| `superuser` | 默认只有超级用户能 `CREATE EXTENSION` |
| `trusted` 扩展 | 标记为 trusted 的可由有 `CREATE` 权限的普通用户安装(如 `pg_trgm`、`hstore`) |
| `SECURITY DEFINER` 函数 | 扩展提供的此类函数是提权风险点 |
| `search_path` | 扩展函数内若依赖 `search_path`,可被劫持 → 应在函数上固定 `search_path` |

```sql
-- 查看哪些扩展可信
SELECT name, default_version, trusted FROM pg_available_extensions WHERE trusted;
```

### FDW 框架(外部数据包装器)

FDW 是**最实用的扩展类型之一**:让 PostgreSQL 把外部数据源当作本地表查询。

```
SQL 查询
   ↓
FDW 接口(postgres_fdw / file_fdw / oracle_fdw ...)
   ↓
外部数据源(另一个 PG 实例 / CSV 文件 / Oracle / MySQL / S3)
```

**五个必要对象**

```sql
-- 1) 扩展
CREATE EXTENSION postgres_fdw;

-- 2) 外部服务器
CREATE SERVER remote_srv FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'remote', port '5432', dbname 'other', fetch_size '10000');

-- 3) 用户映射(本地角色 → 远端凭据)
CREATE USER MAPPING FOR app_user SERVER remote_srv
    OPTIONS (user 'remote_user', password 'secret');

-- 4) 外部表
CREATE FOREIGN TABLE remote_orders (
    id      bigint,
    user_id bigint,
    total   numeric(12,2),
    status  text
) SERVER remote_srv
  OPTIONS (schema_name 'app', table_name 'orders');

-- 5) 使用
SELECT * FROM remote_orders WHERE user_id = 1;
```

**FDW 的核心性能问题:条件下推(pushdown)**

```
理想(下推):
   本地发送:SELECT ... FROM app.orders WHERE user_id = 1
   远端只返回匹配行

糟糕(未下推):
   本地发送:SELECT ... FROM app.orders
   远端返回全表 → 本地再过滤
```

```sql
-- 验证是否下推
EXPLAIN (VERBOSE) SELECT * FROM remote_orders WHERE user_id = 1;
-- 输出中会列出 "Remote SQL: SELECT ... FROM app.orders WHERE (user_id = 1)"
-- 若 Remote SQL 里没有 WHERE,说明未下推
```

**影响下推的因素**

| 因素 | 说明 |
|------|------|
| 表达式是否可下推 | 用了远端不支持的函数/操作符 → 不下推 |
| 排序与聚合 | `ORDER BY` / `GROUP BY` / `LIMIT` 在支持时可下推(PG 会尽量) |
| 连接 | **跨本地与远端表的 JOIN 无法下推**;远端两表 JOIN 可下推 |
| 扩展依赖 | 远端用了 `postgis` 等扩展时,本地需声明 |
| `use_remote_estimate` | 让远端提供真实估算,代价高但更准 |
| `fetch_size` | 每次取的行数 |

```sql
-- 常见优化配置
ALTER SERVER remote_srv OPTIONS (ADD use_remote_estimate 'true');
ALTER SERVER remote_srv OPTIONS (ADD extensions 'postgis');

-- 表级覆盖
ALTER FOREIGN TABLE remote_orders OPTIONS (ADD use_remote_estimate 'true');

-- 查看
SELECT * FROM pg_foreign_server;
SELECT * FROM pg_user_mappings;
SELECT * FROM pg_foreign_table;
```

**本地统计信息(重要)**

优化器默认不知道远端表的数据分布,可能选错计划。

```sql
-- 手工导入远端统计
ANALYZE remote_orders;
-- postgres_fdw 支持 ANALYZE 外部表(可下推到远端)
```

**批量操作与事务**

| 能力 | 说明 |
|------|------|
| `INSERT`/`UPDATE`/`DELETE` | `postgres_fdw` 支持 |
| 批量插入 | `batch_size` 选项控制 |
| 事务 | 本地事务会传给远端;PG18 起 `READ ONLY`/`DEFERRABLE` 状态也会传递 → 只读事务不能再改远端行 |
| 两阶段提交 | 需 `postgres_fdw` 的远端支持 prepared transaction |

```sql
-- 事务传播验证(PG18)
BEGIN READ ONLY;
UPDATE remote_orders SET total = 0 WHERE id = 1;   -- 现在会失败
ROLLBACK;
```

### 可用的 FDW 生态

| FDW | 数据源 |
|-----|--------|
| `postgres_fdw` | 其他 PostgreSQL(内置贡献模块) |
| `file_fdw` | CSV / 文本文件(内置贡献模块) |
| `dblink` | 老式跨库连接(非 FDW,但类似用途) |
| `oracle_fdw` / `mysql_fdw` | 异构数据库 |
| `sqlite_fdw` / `mongo_fdw` | 文件/文档数据库 |
| `parquet_s3_fdw` | 对象存储上的 Parquet |
| `tds_fdw` | SQL Server |
| `citus` / `pg_duckdb` | 分布式 / 分析加速 |

```sql
-- file_fdw:把 CSV 当表
CREATE EXTENSION file_fdw;
CREATE SERVER csv_srv FOREIGN DATA WRAPPER file_fdw;
CREATE FOREIGN TABLE csv_users (
    id int, email text, name text
) SERVER csv_srv OPTIONS (filename '/data/users.csv', format 'csv', header 'true');

SELECT count(*) FROM csv_users;
```

### 扩展的运维

```sql
-- 查看已装与可用
\dx
SELECT * FROM pg_extension;

-- 升级扩展
ALTER EXTENSION pg_trgm UPDATE;

-- 查看扩展包含的对象
\dx+ pg_trgm

-- 移动 schema(relocatable 的扩展)
ALTER EXTENSION pg_trgm SET SCHEMA utils;
```

**`pg_upgrade` 时的注意事项**

| 事项 | 说明 |
|------|------|
| 需在新版本安装相同扩展 | 否则 `--check` 会失败 |
| `shared_preload_libraries` 需确认兼容 | 老扩展可能不支持新内核 |
| 扩展版本需匹配 | 有些扩展为每个 PG 大版本提供不同版本 |
| 升级后建议 `ALTER EXTENSION ... UPDATE` | 获得该版本的新能力 |

> **PG18 的 `pg_upgrade` 会做兼容性检查**,包括扩展与数据校验和设置。升级前先在测试环境跑 `--check`。

## ③ 源码路径

| 内容 | 位置 |
|------|------|
| 扩展加载 | `src/backend/commands/extension.c` |
| 动态库加载 | `src/backend/utils/fmgr/dfmgr.c` |
| 钩子定义 | `src/include/tcop/utility.h`(`ProcessUtility_hook_type`)等 |
| 执行器钩子 | `src/include/executor/executor.h` |
| 后台 worker | `src/backend/postmaster/bgworker.c`, `src/include/postmaster/bgworker.h` |
| FDW 接口 | `src/include/foreign/fdwapi.h` |
| `postgres_fdw` | `contrib/postgres_fdw/` |
| `file_fdw` | `contrib/file_fdw/` |
| `pg_stat_statements` | `contrib/pg_stat_statements/` |
| 扩展目录 | `src/include/catalog/pg_extension.h` |

## ④ 实验验证

### 实验 1:确认扩展的预加载要求

```sql
-- 未预加载时:报错
CREATE EXTENSION pg_stat_statements;
-- ERROR: pg_stat_statements must be loaded via "shared_preload_libraries"

-- 已预加载时可以创建
SELECT name, installed_version FROM pg_available_extensions
WHERE name IN ('pg_stat_statements','pg_trgm','pgcrypto');

SHOW shared_preload_libraries;
```

### 实验 2:验证可信扩展

```sql
SELECT name, trusted FROM pg_available_extensions
WHERE name IN ('pg_trgm','hstore','pgcrypto','pg_stat_statements','postgis');

-- 建一个非超级用户,尝试安装可信/不可信扩展
CREATE ROLE ext_tester LOGIN PASSWORD 'x' CREATEDB;
-- 以 ext_tester 身份:
-- CREATE EXTENSION pg_trgm;          ✅ trusted
-- CREATE EXTENSION pg_stat_statements; ❌ 需超级用户
```

### 实验 3:file_fdw 读 CSV

```bash
# 造一个 CSV 到容器内
docker exec pg18 bash -c "printf 'id,email,name\n1,a@b.c,Ann\n2,c@d.e,Ben\n' > /tmp/u.csv"
```

```sql
CREATE EXTENSION IF NOT EXISTS file_fdw;
CREATE SERVER csv_srv FOREIGN DATA WRAPPER file_fdw;
CREATE FOREIGN TABLE csv_users (id int, email text, name text)
    SERVER csv_srv OPTIONS (filename '/tmp/u.csv', format 'csv', header 'true');

SELECT * FROM csv_users;

-- 观察计划:file_fdw 支持条件下推吗?
EXPLAIN (VERBOSE, ANALYZE) SELECT * FROM csv_users WHERE id = 1;
-- 关注是否有 Filter（可能不下推，全文件扫描）
```

### 实验 4:postgres_fdw 条件下推验证

```bash
# 复用流复制实验中的 5433 实例,或直接在本实例做 self-fdw
docker exec pg18 psql -U postgres -c "
CREATE EXTENSION IF NOT EXISTS postgres_fdw;
CREATE SERVER self_srv FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'localhost', port '5432', dbname 'postgres');
CREATE USER MAPPING FOR postgres SERVER self_srv
  OPTIONS (user 'postgres', password 'postgres');
CREATE FOREIGN TABLE fdw_orders (id bigint, user_id bigint, total numeric(12,2), status text)
  SERVER self_srv OPTIONS (schema_name 'public', table_name 'orders');
"
```

```sql
-- 下推验证
EXPLAIN (VERBOSE) SELECT * FROM fdw_orders WHERE user_id = 1;
-- 看 "Remote SQL:" 是否含 WHERE

-- 不下推的情况(远端不支持的函数)
EXPLAIN (VERBOSE) SELECT * FROM fdw_orders WHERE regexp_replace(status,'a','b') = 'x';
-- Remote SQL 可能不带该条件 → 全量拉取后本地过滤

-- 排序/限制下推
EXPLAIN (VERBOSE) SELECT * FROM fdw_orders ORDER BY id LIMIT 10;
-- 期望 Remote SQL 含 ORDER BY 与 LIMIT
```

### 实验 5:跨本地与远端 JOIN 的性能陷阱

```sql
-- 本地表
CREATE TABLE local_u (id bigint PRIMARY KEY, name text);
INSERT INTO local_u SELECT g, 'u'||g FROM generate_series(1, 1000) g;
ANALYZE local_u;

-- 本地 JOIN 远端：远端表会被全量拉取
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.name, o.total
FROM local_u u JOIN fdw_orders o ON o.user_id = u.id
WHERE u.id < 10;
-- 关注远端返回行数(Foreign Scan 的 rows)与耗时
```

### 实验 6:本地统计对计划的影响

```sql
-- 无统计
TRUNCATE fdw_orders;   -- 不适用外部表,跳过
-- 直接对比
EXPLAIN SELECT * FROM fdw_orders;
-- 默认估算 rows 可能很粗(如 1000)

ANALYZE fdw_orders;   -- postgres_fdw 支持 ANALYZE
EXPLAIN SELECT * FROM fdw_orders;
-- 估算应更贴近真实行数
```

### 实验 7:观察扩展安装的对象

```sql
\dx+ pg_trgm
-- 列出该扩展创建的:函数、操作符、操作符类、类型等

-- 用系统目录查看
SELECT e.extname, d.classid::regclass, d.objid, d.objsubid
FROM pg_extension e
JOIN pg_depend d ON d.refobjid = e.oid AND d.refclassid = 'pg_extension'::regclass
WHERE e.extname = 'pg_trgm' LIMIT 20;
```

## ⑤ 生产启示

| 结论 | 行动 |
|------|------|
| 预加载型扩展必须重启 | 部署流程要区分"需要重启的扩展" |
| 扩展影响升级可行性 | `pg_upgrade --check` 先验证;提前确认扩展有新版本 |
| 只装可信来源的扩展 | C 扩展可进内核,风险等同代码执行 |
| FDW 的性能命门是下推 | 用 `EXPLAIN VERBOSE` 检查 `Remote SQL`;必要时开 `use_remote_estimate` |
| 跨本地/远端 JOIN 很贵 | 尽量让过滤与连接都在远端完成 |
| FDW 需本地统计 | 定期 `ANALYZE` 外部表 |
| `search_path` 要固定 | 扩展函数(尤其 `SECURITY DEFINER`)必须固定 `search_path` |
| 优先用 `trusted` 扩展 | 减少超级用户依赖 |
| 扩展数量要克制 | 每个扩展都增加升级、备份、安全审计的负担 |

```sql
-- 扩展清单与预加载对照
SELECT e.extname, e.extversion,
       (SELECT setting FROM pg_settings WHERE name='shared_preload_libraries') AS preload
FROM pg_extension e ORDER BY 1;

-- FDW 对象总览
SELECT s.srvname, w.fdwname, s.srvoptions FROM pg_foreign_server s
JOIN pg_foreign_data_wrapper w ON w.oid = s.srvfdw;
SELECT ft.relname AS foreign_table, s.srvname
FROM pg_foreign_table t
JOIN pg_class ft ON ft.oid = t.ftrelid
JOIN pg_foreign_server s ON s.oid = t.ftserver;
```

## 相关页面

- [[postgresql/02-中段阶段-专题精讲/扩展生态/专题概览]]
- [[postgresql/02-中段阶段-专题精讲/备份恢复与高可用/升级与迁移]]
- [[postgresql/03-进阶阶段-深度剖析/查询优化器/原理剖析-优化阶段与代价模型]]
