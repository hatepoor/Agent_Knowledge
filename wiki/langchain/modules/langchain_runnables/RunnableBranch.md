---
tags: [langchain, runnables, branch, conditional]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# RunnableBranch

## 是什么

`RunnableBranch` 实现**条件分支**逻辑，根据输入动态选择执行不同的 Runnable 分支，类似于编程中的 `if-elif-else`。

## 导入

```python
from langchain_core.runnables import RunnableBranch, RunnableLambda
```

## 用法

```python
from langchain_core.runnables import RunnableBranch, RunnableLambda
from langchain_core.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser

model = init_chat_model("openai:gpt-5.5")

# 条件判断函数
is_question = RunnableLambda(lambda x: "?" in x["text"])

# 不同分支的处理链
answer_q = ChatPromptTemplate.from_messages([
    ("system", "You are a Q&A assistant."),
    ("human", "{text}"),
]) | model | StrOutputParser()

describe = ChatPromptTemplate.from_messages([
    ("system", "Describe the topic."),
    ("human", "{text}"),
][[) | model | StrOutputParser()

# 构建条件分支
branch = RunnableBranch(
    (is_question, answer_q),                                  # 如果是问题 → answer_q
    (RunnableLambda(lambda x: True), describe),               # 默认分支 → describe
)

print(branch.invoke({"text": "What is AI?"}))   # 走 answer_q
print(branch.invoke({"text": "Artificial Intelligence"}))  # 走 describe
```

## 结构

```python
RunnableBranch(
    (condition_1, runnable_1),    # 条件1为 True → 执行 runnable_1
    (condition_2, runnable_2),    # 条件2为 True → 执行 runnable_2
    (default_condition, default_runnable),  # 默认分支
)
```

- 条件按顺序检查，第一个为 True 的条件对应的 Runnable 被执行
- 通常最后一个条件设为 `lambda x: True` 作为默认分支

## 执行流程

```
输入 → condition_1? → Yes → runnable_1
                   → No  → condition_2? → Yes → runnable_2
                                        → No  → default_runnable
```

## 特点

- 条件函数返回 bool 值
- 按顺序检查条件，匹配第一个 True
- 必须有一个默认分支
- 条件和分支都必须是 Runnable

## 何时使用

- 根据输入类型选择不同的处理逻辑
- 路由（如：问题走 QA 链、闲聊走聊天链）
- 多语言路由
- 任何需要条件判断的链式场景

## 相关

- [[langchain/modules/langchain_runnables/概述|← 返回 runnables 概述]]
- [[langchain/modules/langchain_runnables/RunnableParallel|RunnableParallel - 并行执行]]
- [[langchain/modules/langchain_runnables/RunnableLambda|RunnableLambda - 自定义逻辑]]
