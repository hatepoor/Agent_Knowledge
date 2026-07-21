---
tags: [langchain, runnables, passthrough, lcel]
type: concept
created: 2026-07-07
updated: 2026-07-13
langchain_version: ">=0.3.0"
---

# RunnablePassthrough

## 是什么

`RunnablePassthrough` 是 LCEL 中的"恒等函数"——输入原样透传到输出。但它有三个核心能力让它在实际开发中不可或缺：

1. **`.assign()`** — 往输入 dict 上追加计算字段（最常用）
2. **构造时传 callable** — 透传数据的同时执行副作用（如日志、校验）
3. **`.pick()`** — 从 dict 中筛选字段

## 导入

```python
from langchain_core.runnables import RunnablePassthrough
```

---

## 三种核心用法

### 1. 纯透传 — 什么也不改

等价于 `lambda x: x`，但作为 Runnable 可以塞进 LCEL 管道：

```python
passthrough = RunnablePassthrough()
passthrough.invoke({"a": 1, "b": 2})  # → {"a": 1, "b": 2}
```

### 2. 构造时传 callable — 透传 + 副作用

**输入照常透传**，但每次调用会触发你的函数（日志、校验、上报等）。函数**不应修改返回值**——它的返回值会被忽略。

```python
from langchain_core.runnables import RunnablePassthrough

def log_input(data):
    print(f"[LOG] 收到输入: {data}")

passthrough = RunnablePassthrough(func=log_input)
# invoke 触发了日志，但数据原样输出
result = passthrough.invoke({"query": "今天天气"})
# [LOG] 收到输入: {'query': '今天天气'}
# result = {"query": "今天天气"}
```

也可以用于带校验的透传：

```python
def validate_dict(data):
    if not isinstance(data, dict):
        raise TypeError(f"期望 dict，实际 {type(data)}")
    if "question" not in data:
        raise ValueError("输入必须包含 'question' 字段")

safe_passthrough = RunnablePassthrough(func=validate_dict)
```

### 3. `.assign()` — 追加计算字段（最核心用法）

输入是一个 dict，`.assign()` 保留原有字段，并添加新的计算字段。

**参数可以是三种类型**：

| 参数类型 | 示例 | 说明 |
|----------|------|------|
| `Callable[[dict], Any]` | `lambda x: len(x["text"])` | 接收整个输入 dict，返回新字段的值 |
| `Runnable` | `ChatOpenAI()` 或另一个 chain | 接收整个输入 dict，执行后返回值 |
| `Mapping[str, ...]` | `{"a": func1, "b": func2}` | 一次分配多个字段 |

```python
from langchain_core.runnables import RunnablePassthrough

# 单个 lambda 字段
chain = RunnablePassthrough.assign(
    word_count=lambda x: len(x["text"].split())
)
chain.invoke({"text": "hello world"})
# → {"text": "hello world", "word_count": 2}

# 传入 Runnable（例如另一个模型调用）
from langchain_deepseek import ChatDeepSeek
summarizer = ChatDeepSeek(model="deepseek-v4-flash")

chain = RunnablePassthrough.assign(
    summary=lambda x: summarizer.invoke(f"总结: {x['text']}")
)

# 一次分配多个字段
chain = RunnablePassthrough.assign(
    word_count=lambda x: len(x["text"].split()),
    is_short=lambda x: len(x["text"]) < 100,
)
chain.invoke({"text": "hello"})
# → {"text": "hello", "word_count": 1, "is_short": True}
```

---

## 实战模式

### 模式 1：经典 RAG — 检索上下文 + 保留原问题

这是 RunnablePassthrough 最广泛的用法。用户输入只有问题，但 prompt 需要 `{question, context}` 两个字段：

```python
from langchain_core.runnables import RunnablePassthrough

rag_chain = (
    # 输入: "where did harrison work?"
    # 这一步同时运行: 
    #   context  → retriever.invoke("where did harrison work?") 
    #   question → RunnablePassthrough() 原样透传
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)

rag_chain.invoke("where did harrison work?")
```

**执行流程**：
```
"where did harrison work?"
    → {"context": [Doc(...)], "question": "where did harrison work?"}
        → prompt.format(context=..., question=...)
            → 模型调用 → 回答
```

### 模式 2：并行处理 + 透传（与 RunnableParallel 配合）

原始数据需要同时被多个分支处理：

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

parallel_chain = RunnableParallel(
    # passed 原样透传整个输入
    passed=RunnablePassthrough(),
    # modified 只取 num 字段并 +1
    modified=lambda x: x["num"] + 1,
)

parallel_chain.invoke({"num": 1})
# → {"passed": {"num": 1}, "modified": 2}
```

### 模式 3：链式 assign — 逐步丰富数据

每次 assign 基于前一步的完整 dict 计算新字段：

```python
chain = (
    RunnablePassthrough.assign(
        docs=lambda x: retriever.invoke(x["question"])
    )
    .assign(
        context=lambda x: "\n".join(d.page_content for d in x["docs"])
    )
    .assign(
        answer=lambda x: model.invoke(
            f"根据:{x['context']}\n回答:{x['question']}"
        )
    )
)

# 输入: {"question": "..."}
# 输出: {"question": "...", "docs": [...], "context": "...", "answer": "..."}
```

### 模式 4：多链串联 — 中间步骤中透传原始数据

一个链的输出传给下一个链，但下一个链还需要第一个链的原始输入：

```python
from operator import itemgetter

# 链1: 生成颜色
color_generator = (
    {"attribute": RunnablePassthrough()}
    | prompt1 | model | {"color": StrOutputParser()}
)

# 链2: 拿到颜色后，同时查水果和国家
# itemgetter 用来从 dict 中精准取字段
question_generator = (
    color_generator
    | {"fruit": color_to_fruit, "country": color_to_country}
    | prompt4
)

question_generator.invoke("warm")
# 内部流程: "warm" → color → strawberry/China → "水果是啥颜色，国旗是啥颜色？"
```

### 模式 5：`.pick()` 筛选字段

当一个链返回了多个字段，但下游只需要其中一部分：

```python
from langchain_core.runnables import RunnablePassthrough

full_chain = RunnablePassthrough.assign(
    a=lambda x: 1,
    b=lambda x: 2,
    c=lambda x: 3,
)

# 只要 a 和 b
filtered = full_chain.pick(["a", "b"])
filtered.invoke({"x": 0})
# → {"a": 1, "b": 2}
```

---

## RunnablePassthrough vs RunnableLambda

| | RunnablePassthrough | RunnableLambda |
|---|---|---|
| **本质** | 恒等函数（透传） | 任意函数（变换） |
| **输入** | 原样输出 | 经过函数变换后输出 |
| **assign 语义** | 追加字段，不改变原有字段 | 需要手动 merge 原输入 |
| **使用场景** | RAG 流水线、数据增强 | 数据清洗、格式转换 |

```python
from langchain_core.runnables import RunnablePassthrough, RunnableLambda

# RunnablePassthrough: 输入是什么，输出就是什么 (didn't change input)
passthrough = RunnablePassthrough()
passthrough.invoke({"a": 1})  # → {"a": 1}

# RunnableLambda: 输入被函数转换了
double = RunnableLambda(lambda x: x * 2)
double.invoke(5)  # → 10
```

---

## 常见陷阱

### 陷阱 1：修改原输入导致数据污染

`assign` 要求传入**纯函数**，返回一个新的值，不应该修改原 dict：

```python
# ❌ 错误: 直接修改了输入 dict
def bad_func(x):
    x["extra"] = "数据污染"  # 副作用!
    return x
RunnablePassthrough.assign(bad_result=bad_func)

# ✅ 正确: 返回新值，不碰原 dict
RunnablePassthrough.assign(
    good_result=lambda x: f"基于 {x['query']} 计算的结果"
)
```

### 陷阱 2：类型不匹配

如果输入不是 dict，`.assign()` 会报错。只能对 dict 输入使用 assign。

```python
passthrough = RunnablePassthrough.assign(count=lambda x: 1)
passthrough.invoke("hello")  
# ❌ AttributeError: 'str' object has no attribute 'keys'

# 解决办法：先转成 dict
from langchain_core.runnables import RunnableLambda
chain = RunnableLambda(lambda x: {"text": x}) | RunnablePassthrough.assign(count=...)
```

### 陷阱 3：多个 assign 拆太细

```python
# ❌ 过于冗长
chain = (
    RunnablePassthrough.assign(a=func1)
    .assign(b=func2)
    .assign(c=func3)
    .assign(d=func4)
)

# ✅ 一次 assign 用 Mapping 传入多个字段
chain = RunnablePassthrough.assign(
    a=func1,
    b=func2,
    c=func3,
    d=func4,
)
```

### 陷阱 4：构造时 callable 的返回值会被忽略

```python
def my_func(data):
    return "这个返回值会被丢弃!"

p = RunnablePassthrough(func=my_func)
p.invoke({"a": 1})  # → {"a": 1}，不是 "这个返回值会被丢弃!"
```

`func` 只做副作用（日志、校验），**输出永远是输入**。

---

## 子页面

- [[langchain/modules/langchain_runnables/RunnableLambda|RunnableLambda - 自定义转换]]
- [[langchain/modules/langchain_runnables/RunnableParallel|RunnableParallel - 并行执行]]

## 相关

- [[langchain/modules/langchain_runnables/概述|← 返回 runnables 概述]]
- [官方文档：如何在步骤之间传递参数](http://python.langchain.ac.cn/docs/how_to/passthrough/)
- [官方 API：RunnablePassthrough](https://reference.langchain.com/python/langchain-core/runnables/passthrough/RunnablePassthrough)
