---
tags: [concept, Python, SQLModel, 数据库引擎, Session, 连接池]
type: concept
created: 2026-07-31
updated: 2026-07-31
---

# SQLModel 引擎与 Session

> `create_engine` 来自 SQLAlchemy 核心，是连接数据库的**入口对象**；`Session` 是执行 ORM 操作（增删改查、事务）的**工作单元**。SQLModel 直接复用这两者，本页覆盖生产级引擎配置与 Session 管理。

## 完整代码（引擎工厂）

```python
from typing import Any

from sqlalchemy import event
from sqlalchemy.engine import Engine
from sqlmodel import create_engine
from sqlmodel.pool import StaticPool

def _enable_sqlite_foreign_keys(
        dbapi_connection: Any,
        _: Any,
) -> None:
    cursor = dbapi_connection.cursor()
    cursor.execute("PRAGMA foreign_keys=ON")
    cursor.close()

def create_db_engine(
        database_url: str,
        *,
        echo: bool = False,
) -> Engine:
    if database_url in {"sqlite://", "sqlite:///:memory"}:
        engine = create_engine(
            database_url,
            echo=echo,
            connect_args={"check_same_thread": False},
            poolclass=StaticPool,
        )
    else:
        engine = create_engine(
            database_url,
            echo=echo,
            connect_args={"check_same_thread": False},
        )
    event.listen(
        engine,
        "connect",
        _enable_sqlite_foreign_keys,
    )

    return engine
```

这个工厂函数做了三件事：**创建引擎 → 内存库特殊处理（StaticPool）→ 注册事件自动开外键**。

---

## 一、导入与参数详解

### 1.1 每个导入的作用

| 导入 | 作用 |
|------|------|
| `from typing import Any` | 类型提示的"任意类型"标记。数据库原生连接对象类型无法精确标注（可能是 sqlite3/psycopg2 等），用 `Any` 跳过检查 |
| `from sqlalchemy import event` | SQLAlchemy **事件系统**——给引擎注册钩子函数（类似观察者模式） |
| `from sqlalchemy.engine import Engine` | 引擎类的类型标注，用于函数返回类型 `-> Engine` |
| `from sqlmodel import create_engine` | 创建引擎的函数。SQLModel 直接复用 SQLAlchemy 的 `create_engine`（同函数，兼容 ORM 定义） |
| `from sqlmodel.pool import StaticPool` | **静态连接池**：整个应用永远只创建 1 个连接，反复复用 |

### 1.2 连接串 `database_url` 格式

```python
# SQLite 内存库（测试用，重启数据清空）
"sqlite://"               # 等价于下面这种
"sqlite:///:memory:"

# SQLite 磁盘文件
"sqlite:///./data/app.db"

# PostgreSQL
"postgresql://user:pass@host:5432/dbname"

# MySQL
"mysql+pymysql://user:pass@host:3306/dbname"
```

**结构拆解**：`协议://用户:密码@主机:端口/数据库名`

---

## 二、`create_db_engine` 函数参数

### 2.1 `database_url: str` — 连接串（必填）

见上文 1.2，决定连接什么数据库。

### 2.2 `*` — 仅限关键字参数标记

`*` 之后的所有参数**必须用关键字传**，不能按位置传：

```python
create_db_engine("sqlite:///data.db", True)   # ❌ 报错：echo 必须用关键字
create_db_engine("sqlite:///data.db", echo=True)  # ✅
```

作用：防止调用时参数顺序搞混，提高可读性。

### 2.3 `echo: bool = False` — 是否打印 SQL

```python
engine = create_db_engine("sqlite:///data.db", echo=True)
# 执行任何 SQL 时控制台会输出：
# 2026-07-31 17:00:00 INFO sqlalchemy.engine.Engine SELECT * FROM users ...
```

| 值 | 效果 |
|----|------|
| `False`（默认） | 不打印，生产环境推荐 |
| `True` | 打印所有执行的 SQL 语句，调试阶段很有用 |

---

## 三、`create_engine` 参数详解

### 3.1 `connect_args={"check_same_thread": False}` — 解除 SQLite 线程限制

SQLite 默认规定：**连接只能被创建它的线程使用**。FastAPI 是多线程处理请求的，跨线程用同一个连接会报错：

```
SQLite objects created in a thread can only be used in that same thread
```

`check_same_thread: False` 告诉 SQLite：允许这个连接被其他线程使用（SQLAlchemy 内部有连接锁保证安全）。

**只对 SQLite 有效**，PostgreSQL/MySQL 不需要这个参数。

### 3.2 `poolclass=StaticPool` — 静态连接池（内存库专属）

**为什么内存库必须用 StaticPool**——SQLite 内存库的致命特性：

```
":memory:" 数据库只存在于"单个连接"的内存里
    │
    ├── 连接1: 建表 users、插入数据 → 数据在连接1的内存
    ├── 连接2: 查 users → 表不存在！← 连接2 是另一个独立空库
    │
    ▼
StaticPool 解决: 永远只创建 1 个连接
    → 所有会话复用同一个连接 → 所有操作落在同一个内存库 → 数据可见
```

| 连接池 | 行为 | 适用 |
|--------|------|------|
| 默认（QueuePool） | 用完归还、按需新建连接 | 磁盘文件库 / PostgreSQL / MySQL（多连接没问题） |
| `StaticPool` | **只创建 1 个连接，永远复用** | `:memory:` 内存库（否则每次换连接=换了个空库） |

**其余常用 poolclass**（未在代码中出现，供参考）：

| poolclass | 作用 |
|-----------|------|
| `NullPool` | 每次新建、用完即关，不复用（适合单次脚本） |
| `QueuePool` | 默认池，维护固定大小的连接队列 |
| `AsyncAdaptedQueuePool` | 异步引擎专用 |

---

## 四、`event.listen` 与事件回调

### 4.1 参数详解

```python
event.listen(
    engine,                        # ① 监听对象：哪个引擎
    "connect",                     # ② 事件名：什么时机触发
    _enable_sqlite_foreign_keys,   # ③ 回调函数：触发时执行谁
)
```

| 事件名 | 触发时机 |
|--------|---------|
| `"connect"` | 每次**建立新的数据库连接**时 |
| `"checkout"` | 从连接池取出连接时 |
| `"checkin"` | 连接归还连接池时 |
| `"begin"` / `"commit"` / `"rollback"` | 事务开始/提交/回滚时 |

### 4.2 回调函数签名

```python
def _enable_sqlite_foreign_keys(
        dbapi_connection: Any,   # ① 原生连接对象（如 sqlite3.Connection）
        _: Any,                  # ② 连接记录（ConnectionRecord），下划线 = 用不上
) -> None:
```

`"connect"` 事件的回调固定接收 **2 个参数**：`(dbapi_connection, connection_record)`。即使不需要第二个参数，也必须声明（所以用 `_` 占位）。

### 4.3 为什么需要手动开外键

> **SQLite 默认不强制外键约束**（为了向后兼容）。就算表定义了 `FOREIGN KEY`，插入违反约束的数据也不会报错。必须在**每个连接**上执行 `PRAGMA foreign_keys=ON` 才生效。

而 `PRAGMA` 设置是**连接级**的——每开一个新连接都要重新设置，所以才用 `event.listen("connect")` 自动对所有新连接执行。

**事件执行时序**：

```
event.listen(engine, "connect", handler)   ← 只是"登记"，此时 handler 不执行！
    │
    ▼ （之后任何地方第一次用引擎）
engine.connect() / 首次执行 SQL
    │
    ▼
SQLAlchemy 建立新连接
    │
    ▼ "connect" 事件触发
handler(新连接, 记录) → 执行 PRAGMA foreign_keys=ON
    │
    ▼
连接交给会话正常使用
```

---

## 五、Session 详解

### 5.1 Session 是什么

`Session` 是 SQLModel/SQLAlchemy 的**工作单元**：一次会话 = 一个数据库事务的边界。所有增删改查都在 Session 内进行。

```python
from sqlmodel import Session

with Session(engine) as session:
    # ① 进入会话：事务开始（自动 BEGIN）
    hero = Hero(name="Deadpond", secret_name="Dive Wilson")
    session.add(hero)          # ② 暂存到会话（未入库）
    session.commit()           # ③ 提交：事务 COMMIT，真正写入
# ④ with 退出：关闭连接（未 commit 则自动回滚）
```

### 5.2 Session 生命周期（数据流向）

```
with Session(engine) as session:
    │
    ├── 进入：从连接池取连接，BEGIN 事务
    │
    ├── session.add(obj)      → 对象进入会话"待处理"区（内存）
    ├── 修改对象属性          → 被会话跟踪（change tracking）
    │
    ├── session.commit()      → 发送所有 INSERT/UPDATE/DELETE → COMMIT
    │                              → 事务结束，开启新事务
    ├── session.refresh(obj)  → 回查数据库，刷新对象最新值
    │
    └── with 退出：close()    → 未提交的改动自动 ROLLBACK
```

**关键**：`with` 退出**不会自动 commit**——只负责关闭连接。忘记 `commit()` 是数据丢失的头号原因。

### 5.3 Session 方法速查

| 方法 | 作用 |
|------|------|
| `session.add(obj)` | 把对象加入会话（新建/更新时用） |
| `session.add_all([objs])` | 批量加入 |
| `session.commit()` | 提交事务，写入所有挂起改动 |
| `session.refresh(obj)` | 重新从数据库读取对象状态 |
| `session.delete(obj)` | 标记对象删除（commit 后生效） |
| `session.exec(stmt)` | 执行查询（SQLModel 特色，返回模型实例） |
| `session.rollback()` | 回滚当前事务，放弃未提交改动 |
| `session.close()` | 关闭会话，归还连接 |

### 5.4 FastAPI 中的 Session 注入（推荐模式）

```python
from fastapi import Depends, FastAPI
from sqlmodel import Session

app = FastAPI()

def get_session():
    """依赖注入：每个请求一个 Session"""
    with Session(engine) as session:
        yield session

@app.get("/users/{user_id}")
def get_user(user_id: int, session: Session = Depends(get_session)):
    return session.exec(select(User).where(User.id == user_id)).first()
```

**每个请求独立 Session**：请求进来开新事务，请求结束自动关闭，天然隔离错误、避免连接泄漏。

---

## 六、数据流向总览

```
create_db_engine("sqlite:///:memory:")
    │
    ├── 是内存库？ → poolclass=StaticPool（保证单连接，数据共享）
    ├── 否        → 默认连接池（磁盘文件多连接 OK）
    │
    ├── connect_args 加 check_same_thread=False（FastAPI 多线程安全）
    │
    ├── event.listen 注册 connect 事件 → 新连接自动开外键
    │
    ▼
返回 engine ──→ Session(engine) ──→ CRUD / 查询操作
```

---

## 七、调用示例

```python
# 测试用：内存库（同进程内数据共享，重启清空）
engine = create_db_engine("sqlite:///:memory:", echo=True)

# 生产用：磁盘 SQLite 文件
engine = create_db_engine("sqlite:///./data/app.db")
```

## 相关页面

- [[sqlmodel/概述]] — SQLModel 模块导航
- [[sqlmodel/模型定义]] — 模型与字段定义
- [[sqlmodel/CRUD操作]] — 引擎之上的增删改查
- [[sqlmodel/关系与关联]] — 关系模型配置
- [[python/advanced/Pydantic数据验证]] — SQLModel 依赖的 Pydantic 机制
- [[fastapi/依赖注入]] — FastAPI 中 Session 的依赖注入
- [[tools/向量数据库/pgvector/概述]] — PostgreSQL 向量数据库（也用 SQLAlchemy 引擎）
