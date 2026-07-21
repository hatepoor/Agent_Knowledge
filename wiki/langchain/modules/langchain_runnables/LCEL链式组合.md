---
tags: [langchain, runnables, lcel, chain]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# LCEL 链式组合

## 是什么

LCEL（LangChain Expression Language）使用 `|` 管道操作符将多个 Runnable 组件**串行组合**成一条执行链。`|` 操作符本质上是 `RunnableSequence` 的语法糖。

必须是runnable的子类，或者是callable，mapping的子类对象
## 导入

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain.chat_models import init_chat_model
```

## 基本语法

```python
chain = prompt | model | parser
```

每个组件都是 Runnable，`|` 将它们按顺序串联：前一个的输出作为后一个的输入。

## 完整示例

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser

model = init_chat_model("openai:gpt-5.5")

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a translator to {language}."),
    ("human", "{text}"),
])
chain = prompt | model | StrOutputParser()
print(chain.invoke({"language": "French", "text": "Hello world"}))
```

## 执行流程

```
输入 dict → Prompt（生成 ChatPromptValue）
           → Model（生成 AIMessage）
           → Parser（提取 str）
           → 输出
```

## 调用方式

```python
# 同步调用
result = chain.invoke({"language": "French", "text": "Hello"})

# 异步调用
result = await chain.ainvoke({"language": "French", "text": "Hello"})

# 流式输出
for chunk in chain.stream({"language": "Japanese", "text": "Thank you"}):
    print(chunk, end="", flush=True)

# 批量调用
results = chain.batch([
    {"language": "French", "text": "Yes"},
    {"language": "German", "text": "No"},
    {"language": "Spanish", "text": "Maybe"},
])
```

## RunnableConfig 注入

```python
result = chain.invoke(
    {"language": "German", "text": "Good morning"},
    config={
        "tags": ["translation", "demo"],
        "metadata": {"user_id": "u1"},
        "callbacks": [],
    },
)
```

## 注意事项

- 所有组件必须实现 Runnable 接口
- 前一个组件的输出类型必须兼容后一个组件的输入类型
- `|` 操作符会自动创建 `RunnableSequence`
- 旧式 `LLMChain`、`SequentialChain` 等已弃用

## LCEL 链的返回类型

### invoke() 返回类型

链的 `invoke()` 返回类型由**链中最后一个组件**决定：

| 链的最后一个组件 | `invoke()` 返回类型 |
|-----------------|-------------------|
| `StrOutputParser` | `str` |
| `JsonOutputParser` | `dict` |
| `PydanticOutputParser` | Pydantic `BaseModel` 实例 |
| `ChatModel`（无 parser） | `AIMessage` |
| `PromptTemplate`（无后续） | `StringPromptValue` |
| `ChatPromptTemplate`（无后续） | `ChatPromptValue` |
| `VectorStoreRetriever` | `list[Document][[` |
| `RunnableLambda(fn)` | `fn` 的返回值 |

### stream() 产出类型

| 链的最后一个组件 | `stream()` 产出类型 |
|-----------------|-------------------|
| `StrOutputParser` | `str`（逐 token） |
| `JsonOutputParser` | `dict`（JSON 完整时一次性产出） |
| `ChatModel`（无 parser） | `AIMessageChunk` |

### 完整类型流转示例

```python
chain = prompt | model | parser

# invoke 的类型流转：
# dict → ChatPromptValue → AIMessage → str
result = chain.invoke({"question": "什么是AI？"})
print(type(result))  # <class 'str'>

# stream 的类型流转：
# dict → ChatPromptValue → AIMessageChunk → str (逐 token)
for chunk in chain.stream({"question": "什么是AI？"}):
    print(type(chunk))  # <class 'str'>
    print(chunk, end="", flush=True)
```

## 相关

- [[langchain/modules/langchain_runnables/概述|← 返回 runnables 概述]]
- [[langchain/modules/langchain_runnables/RunnableLambda|RunnableLambda - 自定义步骤]]
- [[langchain/modules/langchain_runnables/RunnableParallel|RunnableParallel - 并行执行]]
