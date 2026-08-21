---
tags: [concept, Python, pytest, fixture, 测试框架, 单元测试]
type: concept
created: 2026-07-31
updated: 2026-07-31
---

# pytest 与 fixture

> pytest 是 Python 最流行的测试框架。`@pytest.fixture` 把函数标记为"测试夹具"——一个可被多个测试共享的**准备/清理组件**，通过函数参数名自动注入给测试函数。

## fixture 解决的问题

测试前往往需要"准备数据/环境"（建数据库连接、创建测试对象、临时文件），测试后还要清理。没有 fixture：

```python
# ❌ 每个测试都重复准备+清理
def test_user_1():
    db = connect_db()          # 准备
    db.create_table("users")
    ...测试逻辑...
    db.drop_table("users")     # 清理
    db.close()

def test_user_2():
    db = connect_db()          # 重复准备！
    db.create_table("users")
    ...测试逻辑...
    db.drop_table("users")     # 重复清理！
    db.close()
```

fixture 把这些准备工作**抽出来复用**，并自动管理清理。

## 一、基本用法

```python
import pytest
from sqlmodel import Session, create_engine

# ① 标记为 fixture
@pytest.fixture
def db_session():
    """每个测试独享的内存数据库会话"""
    engine = create_engine("sqlite:///:memory:", poolclass=StaticPool)
    SQLModel.metadata.create_all(engine)
    with Session(engine) as session:
        yield session        # ② 把 session 交给测试函数
        # ③ 测试结束后回到这里 → 清理

# ④ 测试函数参数名 = fixture 函数名，pytest 自动注入
def test_create_user(db_session):      # ← 参数 db_session 自动拿到 fixture 返回值
    user = User(name="Alice")
    db_session.add(user)
    db_session.commit()
    assert user.id is not None

def test_query_user(db_session):       # ← 每个测试拿到独立的 fixture 实例
    users = db_session.exec(select(User)).all()
    assert len(users) == 0             # 相互隔离，互不污染
```

## 二、执行流程（数据流向）

```
pytest 运行 test_create_user
    │
    ├── ① 发现参数 db_session → 查找同名 fixture
    │
    ├── ② 执行 fixture 函数体（到 yield 暂停）
    │      engine 创建 → 建表 → Session 打开 → yield session
    │
    ├── ③ 把 yield 出来的值注入测试函数参数
    │      test_create_user(db_session=<session 对象>)
    │
    ├── ④ 测试函数执行
    │
    └── ⑤ 测试结束 → 回到 yield 之后继续执行（清理代码）
           session 关闭 → 数据库销毁
```

**关键机制**：`yield` 把 fixture 分成两半——`yield` 前是"准备"，`yield` 后是"清理"。测试函数运行期间，fixture 一直"挂着"。

## 三���作用域（scope）——生命周期控制

```python
@pytest.fixture(scope="function")   # 默认：每个测试函数独立创建（最常用）
@pytest.fixture(scope="module")     # 整个模块共享一个实例（测试之间数据可见）
@pytest.fixture(scope="session")    # 整个测试会话共享一个（如全局数据库连接）
@pytest.fixture(scope="class")      # 每个测试类共享一个
```

| scope | 创建时机 | 适用场景 |
|-------|---------|---------|
| `function`（默认） | 每个测试前 | 数据隔离要求高的场景（每个测试独立数据） |
| `module` | 模块首次用时 | 模块内共享的昂贵资源（如加载一次模型） |
| `session` | 整个测试开始前 | 全局连接、配置加载（如一次性的数据库引擎） |
| `class` | 类首次用时 | 类内共享 |

```python
@pytest.fixture(scope="session")
def engine():
    """整个测试会话只创建一次引擎"""
    return create_engine("sqlite:///test.db")

@pytest.fixture(scope="function")
def session(engine):          # fixture 可以依赖其他 fixture！
    with Session(engine) as s:
        yield s
```

## 四、fixture 之间的依赖

fixture 可以调用另一个 fixture——参数名引用即可，pytest 自动解析依赖链：

```python
@pytest.fixture
def user_data():
    return {"name": "Alice", "age": 30}

@pytest.fixture
def created_user(db_session, user_data):   # 依赖两个 fixture
    user = User(**user_data)
    db_session.add(user)
    db_session.commit()
    return user

def test_user_has_id(created_user):        # 直接依赖 created_user
    assert created_user.id is not None
    # 依赖链：created_user → db_session + user_data（pytest 自动按顺序执行）
```

## 五、常用内置 fixture

```python
def test_tmp(tmp_path):                    # 自动创建临时目录，测试后自动清理
    f = tmp_path / "data.txt"
    f.write_text("hello")
    assert f.read_text() == "hello"

def test_capture(capsys):
    print("输出内容")
    captured = capsys.readouterr()         # 捕获 print 输出
    assert captured.out == "输出内容\n"
```

| 内置 fixture | 作用 |
|-------------|------|
| `tmp_path` / `tmp_path_factory` | 临时目录（自动清理） |
| `capsys` / `capfd` | 捕获 stdout/stderr 输出 |
| `monkeypatch` | 动态修改对象/环境变量（测试后自动还原） |
| `request` | 访问当前测试上下文（参数、fixture 名等） |

## 六、常见误区

```python
# ❌ 只写 yield 前没清理：数据库连接泄漏
@pytest.fixture
def session():
    s = Session(engine)
    yield s
    # 没写 s.close() → 连接泄漏

# ✅ 用 with 自动管理
@pytest.fixture
def session():
    with Session(engine) as s:
        yield s

# ❌ 测试函数没写参数名，fixture 不会自动执行
@pytest.fixture
def data():
    return 42

def test_x():      # 没写 data 参数 → fixture 完全不会跑
    assert data == 42   # NameError！data 未定义

# ✅ 必须声明参数
def test_x(data):
    assert data == 42

# ❌ 修改共享的 session 级 fixture 数据会污染其他测试
```

## 七、与 SQLModel 结合（实战）

```python
import pytest
from sqlmodel import Session, SQLModel, create_engine, select
from sqlmodel.pool import StaticPool

@pytest.fixture(scope="session")
def engine():
    """整个测试会话一个内存引擎"""
    return create_engine(
        "sqlite:///:memory:",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,      # 内存库必须 StaticPool，见引擎与Session
    )

@pytest.fixture
def session(engine):
    """每个测试独立会话 + 建表"""
    SQLModel.metadata.create_all(engine)
    with Session(engine) as s:
        yield s
    # 测试后清空表，隔离测试
    SQLModel.metadata.drop_all(engine)

def test_create_user(session):
    user = User(name="Alice", age=30)
    session.add(user)
    session.commit()
    session.refresh(user)
    assert user.id is not None

def test_empty_db(session):
    users = session.exec(select(User)).all()
    assert users == []   # 上一步的 user 不残留
```

## 相关页面

- [[sqlmodel/引擎与Session]] — 内存库 StaticPool 配置（fixture 中用到）
- [[sqlmodel/CRUD操作]] — 测试的增删改查对象
- [[fastapi/测试]] — FastAPI 应用的 pytest 测试
- [[langgraph/tutorial/测试模式]] — LangGraph 测试模式
- [[python/functions/函数]] — 装饰器基础
