---
tags: [concept, Python, SQLModel, CRUD, Session, 增删改查]
type: concept
created: 2026-07-31
updated: 2026-07-31
---

# SQLModel CRUD 操作

> SQLModel 用 SQLAlchemy 的 `Session` 完成全部增删改查。核心差异：`session.exec(statement)`（SQLModel 特色）替代 SQLAlchemy 的 `session.execute(statement)`——返回对象直接是模型实例，无需手动 `.scalars()` 转换。

## 完整模型（下文示例共用）

```python
from sqlmodel import Field, SQLModel, create_engine, Session, select

class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(index=True)
    secret_name: str
    age: int | None = Field(default=None, index=True)

engine = create_engine("sqlite:///database.db")
SQLModel.metadata.create_all(engine)   # 建表
```

---

## Create — 新增

```python
with Session(engine) as session:
    # ① 创建实例（不提交就不入库）
    hero = Hero(name="Deadpond", secret_name="Dive Wilson")

    # ② 加入会话（staged，此时还在内存）
    session.add(hero)

    # ③ 提交事务（真正写入数据库）
    session.commit()

    # ④ 刷新（拉取数据库生成的值，如自增 id）
    session.refresh(hero)
    print(hero.id)   # 1 ← 刷新前是 None，刷新后才有值
```

**四步流程与数据流向**：

```
Hero(...) 创建实例        ← 纯 Pydantic 校验，未接触数据库
    │
    ▼
session.add(hero)          ← 标记待插入（内存中的工作区）
    │
    ▼
session.commit()           ← 发 INSERT 语句，事务提交
    │
    ▼
session.refresh(hero)      ← 回查数据库，拉回自增 id 等生成值
```

> 不加 `refresh`，`hero.id` 仍是 `None`（虽然数据库已有值）。批量插入时逐条 add 后一次性 commit 即可。

---

## Read — 查询

```python
with Session(engine) as session:
    # 查全部
    statement = select(Hero)
    heroes = session.exec(statement).all()     # list[Hero]

    # 按条件查（精确匹配）
    statement = select(Hero).where(Hero.name == "Deadpond")
    hero = session.exec(statement).first()     # Hero | None

    # 查不到会怎样
    hero = session.exec(statement).first()     # None（不报错）
    hero = session.exec(statement).one()       # 查不到/多条 → 抛异常
```

| 方法 | 返回 | 查不到 | 多条 |
|------|------|--------|------|
| `.first()` | `Hero \| None` | `None` | 第一条 |
| `.one()` | `Hero` | 抛 `NoResultFound` | 抛 `MultipleResultsFound` |
| `.all()` | `list[Hero]` | `[]` | 全部 |

**`session.exec()` 与 `session.execute()` 的区别**：

```python
# SQLAlchemy 风格：返回 Row 结果集，要 .scalars() 转模型
rows = session.execute(select(Hero)).scalars().all()

# SQLModel 风格：直接返回模型列表（等价写法）
heroes = session.exec(select(Hero)).all()
```

---

## Update — 更新

```python
with Session(engine) as session:
    # ① 查出目标对象
    statement = select(Hero).where(Hero.name == "Deadpond")
    hero = session.exec(statement).one()

    # ② 直接改属性（SQLModel 自动跟踪变更）
    hero.age = 42

    # ③ 提交即更新（会自动发 UPDATE，无需手动标记）
    session.add(hero)     # 可省略：已跟踪的对象改动直接生效
    session.commit()
    session.refresh(hero)

    print(hero.age)   # 42
```

**变更跟踪机制**：SQLModel/SQLAlchemy 会话会记录对象的修改。只要对象还在 session 内（已 add 过且未过期），改属性后 `commit()` 就会自动生成 `UPDATE` 语句。

---

## Delete — 删除

```python
with Session(engine) as session:
    statement = select(Hero).where(Hero.name == "Deadpond")
    hero = session.exec(statement).one()

    session.delete(hero)   # 标记删除
    session.commit()       # 提交 → 发 DELETE 语句
```

---

## 完整示例（一次对话全部操作）

```python
def main():
    with Session(engine) as session:
        # Create
        hero = Hero(name="Rusty-Man", secret_name="Tommy Sharp", age=48)
        session.add(hero)
        session.commit()
        session.refresh(hero)
        print("创建:", hero.id, hero.name)

        # Read
        found = session.exec(
            select(Hero).where(Hero.name == "Rusty-Man")
        ).one()
        print("查询:", found.name, found.age)

        # Update
        found.age = 49
        session.add(found)
        session.commit()
        session.refresh(found)
        print("更新:", found.age)

        # Delete
        session.delete(found)
        session.commit()
        print("删除完成，剩余:", len(session.exec(select(Hero)).all()))

if __name__ == "__main__":
    main()
```

## 常见误区

```python
# ❌ 忘记 refresh，id 是 None
session.add(hero)
session.commit()
print(hero.id)    # None ← 需要 refresh

# ❌ 修改后忘记 commit，改动丢失（with 退出不会自动提交）
with Session(engine) as session:
    hero.age = 50
# with 退出只 close 连接，不 commit！改动丢失

# ✅ 需要显式 commit
with Session(engine) as session:
    hero.age = 50
    session.commit()

# ❌ 查不到就 .one()，直接抛异常
hero = session.exec(select(Hero).where(...)).one()   # 可能崩

# ✅ 用 first() 处理可能不存在的情况
hero = session.exec(select(Hero).where(...)).first()
if hero is None:
    print("不存在")
```

## 相关页面

- [[sqlmodel/概述]] — 模块导航
- [[sqlmodel/模型定义]] — 模型与 Field 定义
- [[sqlmodel/查询进阶]] — 过滤/排序/聚合/JOIN
- [[sqlmodel/引擎与Session]] — 引擎配置与会话管理
- [[sqlmodel/引擎与Session|SQLAlchemy]] — 底层机制
