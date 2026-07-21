---
tags: [langchain, runnables, fallback, retry]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# RunnableWithFallbacks

## 是什么

`RunnableWithFallbacks` 实现**容错回退**机制：当主 Runnable 执行失败时，自动切换到备用 Runnable，保证链的可用性。

## 导入

```python
from langchain_core.runnables import RunnableWithFallbacks
# 或通过 .with_fallbacks() 方法
```

## 用法

```python
from langchain.chat_models import init_chat_model

# 主模型和备用模型
primary = init_chat_model("openai:gpt-5.5")
fallback = init_chat_model("openai:gpt-5.4-mini")

# 当 primary 失败时自动用 fallback
robust_model = primary.with_fallbacks([fallback])

# 在链中使用
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("human", "{question}"),
])
chain = prompt | robust_model | StrOutputParser()
result = chain.invoke({"question": "What is LangChain?"})
```

## 多个回退

```python
# 可以设置多个回退，按顺序尝试
model_a = init_chat_model("openai:gpt-5.5")
model_b = init_chat_model("anthropic:claude-sonnet-4")
model_c = init_chat_model("openai:gpt-5.4-mini")

robust_model = model_a.with_fallbacks([model_b, model_c])
# 执行顺序：model_a → 失败 → model_b → 失败 → model_c
```

## 执行流程

```
输入 → primary → 成功 → 输出
                → 失败 → fallback_1 → 成功 → 输出
                                    → 失败 → fallback_2 → 成功 → 输出
                                                        → 失败 → 抛出异常
```

## 特点

- 通过 `.with_fallbacks()` 方法创建，返回新的 Runnable
- 回退列表按顺序尝试
- 所有回退都失败时才抛出异常
- 适用于任何 Runnable（模型、Retriever、Tool 等）

## 与 RunnableRetry 的区别

| 特性 | RunnableWithFallbacks | RunnableRetry |
|------|----------------------|---------------|
| 策略 | 切换到不同的 Runnable | 重试同一个 Runnable |
| 使用 | `.with_fallbacks([...][[)` | `.with_retry(...)` |
| 场景 | 主服务不可用时切换备用 | 临时故障时重试 |

## 何时使用

- 多模型容错（主模型 API 限流或宕机时切换）
- 多数据源容错（主数据库不可用时切换备用）
- 任何需要高可用性的生产环境

## 相关

- [[langchain/modules/langchain_runnables/概述|← 返回 runnables 概述]]
- [[langchain/modules/langchain_runnables/LCEL链式组合|LCEL 链式组合]]
