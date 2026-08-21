---
tags: [concept, Python, functools, 缓存, 装饰器, 高阶函数]
type: concept
created: 2026-07-31
updated: 2026-07-31
---

# functools 函数工具

> `functools` 是 Python **标准库**（内置，无需安装），专门提供**高阶函数**和**函数操作工具**——"操作函数"的函数，如缓存、偏函数、装饰器辅助等。

## 成员一览

| 成员 | 作用 | 一句话 |
|------|------|--------|
| `lru_cache` | 函数结果缓存 | 同参数调用直接返回缓存值，按 LRU 淘汰 |
| `cache` | 简化版 lru_cache | Python 3.9+，无大小限制（等价 `lru_cache(maxsize=None)`） |
| `partial` | 偏函数 | 预先固定部分参数，生成新函数 |
| `wraps` | 装饰器辅助 | 保留被装饰函数的元信息（名字/文档） |
| `reduce` | 累积归约 | 把序列折叠成一个值 |
| `cached_property` | 属性缓存 | 属性只计算一次，后续直接读 |
| `singledispatch` | 单分派泛型函数 | 按第一个参数类型分发不同实现 |

---

## lru_cache — 结果缓存

**LRU（Least Recently Used，最近最少使用）缓存装饰器**：把函数计算结果缓存起来，相同参数再次调用时直接返回缓存，不再重新计算。

### 解决的问题

```python
# 没有缓存：重复计算
def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)

fib(35)  # 第一次算：约 2 秒，递归调用 29,860,703 次
fib(35)  # 第二次算：还是 2 秒！白白重算
```

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)

fib(35)  # 第一次算：仍然慢
fib(35)  # 第二次算：毫秒级，递归调用 36 次（每个 n 只算一次）
```

### 执行流程（数据流向）

```
fib(35) 被调用
    │
    ├── lru_cache 检查缓存: key = (35,)   ← 参数元组作为缓存键
    │       │
    │       ├── 命中 → 直接返回缓存值，函数体不执行
    │       └── 未命中 → 执行函数体
    │               │
    │               └── 计算结果存入缓存: {(35,): 9227465, ...}
    │
    └── 返回结果
```

### 参数详解

```python
@lru_cache(maxsize=128, typed=False)
def expensive(x, y):
    ...
```

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `maxsize` | 128 | 最多缓存多少条结果。**超过后按 LRU 策略淘汰最久未用的**；设为 `None` 表示无上限 |
| `typed` | False | 是否区分类型：`True` 时 `f(3)` 和 `f(3.0)` 视为不同调用（键不同） |

**LRU 淘汰策略**：缓存满了之后，新结果进来时把"最久没被使用"的那条淘汰掉——保留最热门的缓存。

### 注意点

```python
# 1. ❌ 函数参数必须可哈希（hashable）
@lru_cache
def f(items: list):   # list 不可哈希 → 报错
    ...

# ✅ 改传元组
@lru_cache
def f(items: tuple):
    ...

# 2. ⚠️ 有副作用的函数不能缓存（缓存后副作用只执行一次）
@lru_cache
def log_and_compute(x):
    print("计算中...")   # 第二次调用不会打印
    return x * 2

# 3. ✅ 查看/清空缓存
f.cache_info()   # CacheInfo(hits=..., misses=..., maxsize=128, currsize=...)
f.cache_clear()  # 清空缓存
```

### Agent 项目实战：缓存 LLM 实例

```python
from functools import lru_cache
from langchain_deepseek import ChatDeepSeek

@lru_cache(maxsize=8)          # 同参数只创建一次 LLM 实例
def get_llm(model: str):
    return ChatDeepSeek(model=model, api_key=settings.api_key)

# FastAPI 请求里反复调用，但实际只创建 8 种模型的实例
llm = get_llm("deepseek-v4-flash")
```

> FastAPI 的 `@lru_cache` 依赖注入模式与此相同，见 [[fastapi/依赖注入]]。

---

## partial — 偏函数

**预先固定函数的部分参数**，生成一个新函数。适合"同一函数反复传相同参数"的场景。

```python
from functools import partial

def power(base, exponent):
    return base ** exponent

# 固定 exponent=2，生成平方函数
square = partial(power, exponent=2)
# 调用时只需传 base
square(5)   # 25
square(8)   # 64

# 实际使用：反复用同一个 LLM 配置
from langchain_core.prompts import ChatPromptTemplate
system_prompt = partial(
    ChatPromptTemplate.from_messages,
    [("system", "你是助手"), ("human", "{input}")],
)
```

| 对比 | 说明 |
|------|------|
| 直接调用 | `power(5, 2)` 每次都要写全参数 |
| `partial` | 固定 `exponent=2` 后 `square(5)` 只写变化的参数 |
| 与 lambda 区别 | `partial` 保留原函数元信息、可被 pickle 序列化 |

---

## wraps — 装饰器辅助

写装饰器时若不处理，被装饰函数的 `__name__`、`__doc__` 会变成装饰器的。`wraps` 把这些元信息保留下来：

```python
from functools import wraps

def log_call(func):
    @wraps(func)              # ← 把 func 的名字/文档/签名复制到 wrapper 上
    def wrapper(*args, **kwargs):
        print(f"调用 {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log_call
def hello():
    """打印问候"""
    print("你好")

print(hello.__name__)   # "hello"（不加 wraps 会显示 "wrapper"）
print(hello.__doc__)    # "打印问候"（不加 wraps 会是 None）
```

---

## reduce — 累积归约

把序列**折叠成一个值**，按顺序两两应用函数：

```python
from functools import reduce

# 1 + 2 + 3 + 4 + 5
total = reduce(lambda acc, x: acc + x, [1, 2, 3, 4, 5])
# 执行过程：
#   acc=1, x=2 → 3
#   acc=3, x=3 → 6
#   acc=6, x=4 → 10
#   acc=10, x=5 → 15
print(total)   # 15

# 可选初始值（第三个参数）
reduce(lambda acc, x: acc + x, [1, 2, 3], 100)   # 106
```

> 注意：Python 3 起 `reduce` 被移出内置函数，必须从 `functools` 导入。一般可用 `sum()` / 循环替代，仅特殊场景（如嵌套 dict 合并）用 reduce 更简洁。

---

## cached_property — 属性只算一次

`@property` 每次访问都重新计算；`cached_property` 只算一次，之后直接读缓存值：

```python
from functools import cached_property

class Report:
    def __init__(self, data):
        self.data = data

    @cached_property
    def summary(self):
        """只在首次访问时计算一次"""
        print("计算摘要...")     # 只打印一次
        return f"共 {len(self.data)} 条数据"

r = Report([1, 2, 3])
print(r.summary)   # "计算摘要..." → "共 3 条数据"
print(r.summary)   # 直接返回 "共 3 条数据"，不再计算
```

---

## 相关页面

- [[python/advanced/装饰器进阶]] — 装饰器原理与自定义装饰器
- [[python/functions/函数]] — 函数基础与 lambda
- [[python/advanced/Pydantic数据验证]] — pydantic-settings 配置管理（与 lru_cache 常配合）
- [[fastapi/依赖注入]] — FastAPI 中 lru_cache 依赖缓存
