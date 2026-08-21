---
tags: [concept, Python, SimpleNamespace, types, 轻量对象, 测试桩]
type: concept
created: 2026-07-31
updated: 2026-07-31
---

# SimpleNamespace 轻量对象

> `SimpleNamespace`（`from types import SimpleNamespace`）是 Python 标准库 types 模块提供的一个**简单对象容器**——没有自定义方法，属性可以随意增删改查，本质是 `object()` 的升级版（object 不能设属性，它能）。

## 为什么需要它——object() 的局限

```python
# ❌ 普通 object：不能直接设属性
obj = object()
obj.name = "Alice"   # AttributeError: 'object' object has no attribute 'name'

# ✅ SimpleNamespace：可以
obj = SimpleNamespace()
obj.name = "Alice"   # 正常
```

## 三种创建方式

```python
from types import SimpleNamespace

# 方式1：空对象，后续设属性
obj = SimpleNamespace()

# 方式2：构造时直接传关键字参数（最常用）
obj = SimpleNamespace(name="Alice", age=30)

# 方式3：从字典展开（** 拆包）——最常用
data = {"name": "Alice", "age": 30}
obj = SimpleNamespace(**data)
print(obj.name)    # Alice

# 属性可任意增删改查
obj.email = "alice@example.com"   # 添加
del obj.age                        # 删除
```

> **方式3 特别重要**：`SimpleNamespace(**dict)` 一行把字典变成"带属性的对象"。

## 与 dict 的对比

```python
data_dict = {"name": "Alice", "age": 30}
data_ns = SimpleNamespace(name="Alice", age=30)

data_dict["name"]        # Alice（字典：用 [ ] 语法）
data_ns.name             # Alice（命名空间：用 . 语法）
```

| 维度 | `dict` | `SimpleNamespace` |
|------|--------|-------------------|
| 访问语法 | `d["key"]` | `ns.key` |
| 键有非法标识符？ | ✅ 支持（如 `"a b"`） | ❌ 只能标识符属性名 |
| 遍历键 | `d.items()` | `vars(ns).items()` |
| 自动补全 | 无 | ✅ IDE 能提示属性 |
| 序列化 | `json.dumps(d)` | 需先转 `vars(ns)` |

**选择规则**：
- 键名是动态的/非常规标识符 → dict
- 键名固定、想要属性访问的便利 → SimpleNamespace
- 需要 Pydantic 校验 → BaseModel

## 典型使用场景

### 场景1：测试桩（stub）——免定义类，快速造对象

```python
def get_username(user):
    return user.name

# ✅ 测试时直接造个假的，不用为它定义一个 User 类
from types import SimpleNamespace
fake_user = SimpleNamespace(name="Alice")
assert get_username(fake_user) == "Alice"
```

### 场景2：模拟数据库行 / API 响应

```python
# 模拟 SQLAlchemy 查询结果（有 .id / .name 属性）
row = SimpleNamespace(id=1, name="Alice", age=30)
```

### 场景3：字典转对象（配置读取）

```python
config_dict = {"host": "localhost", "port": 8080}
config = SimpleNamespace(**config_dict)
print(config.host)     # localhost —— 比 config["host"] 好读
```

### 场景4：函数返回多值打包

```python
def parse_user(raw):
    return SimpleNamespace(
        name=raw["name"],
        is_admin=raw.get("role") == "admin",
    )

user = parse_user({"name": "Bob"})
print(user.name, user.is_admin)   # Bob False
```

## 与 dataclass / namedtuple 对比

```python
from dataclasses import dataclass
from collections import namedtuple

# SimpleNamespace：轻量，无定义，即用即走
ns = SimpleNamespace(x=1, y=2)

# dataclass：需定义类，但自动生成 __init__/__repr__/__eq__
@dataclass
class Point:
    x: int
    y: int
p = Point(1, 2)

# namedtuple：不可变，且支持索引访问
PointNT = namedtuple("PointNT", ["x", "y"])
pt = PointNT(1, 2)
```

| | `SimpleNamespace` | `dataclass` | `namedtuple` |
|---|---|---|---|
| 需要定义类 | ❌ | ✅ | 用工厂函数 |
| 可变 | ✅ | ✅ | ❌ 不可变 |
| 自动补全 | 一般 | ✅ 强 | ✅ |
| 适用 | 快速桩/临时对象 | 正式业务数据类 | 轻量不可变记录 |

## 小细节

```python
from types import SimpleNamespace

ns = SimpleNamespace(x=1, y=2)

print(ns)                    # namespace(x=1, y=2) ← 自动有 repr
vars(ns)                     # {'x': 1, 'y': 2} ← 转回字典
SimpleNamespace(**vars(ns))  # 复制一份

ns2 = SimpleNamespace(x=1, y=2)
ns == ns2                    # True（属性相同即相等，与 dict 相等性类似）
```

## 相关页面

- [[python/advanced/Pydantic数据验证]] — 需要校验时用 BaseModel 替代
- [[python/testing/pytest与fixture]] — 测试桩的典型使用场景
- [[python/functions/函数]] — 函数返回多值的打包方式
- [[python/data_structures/字典]] — 与 dict 的对比基础
