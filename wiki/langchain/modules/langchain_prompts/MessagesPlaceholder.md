---
tags: [langchain, prompts, MessagesPlaceholder, history, placeholder]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# MessagesPlaceholder

`MessagesPlaceholder` 用于在聊天模板中**插入消息列表**，常用于多轮对话场景。它在模板中充当占位符，运行时会被实际的消息列表替换。

## 基础用法

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage

# 创建占位符
placeholder = MessagesPlaceholder("chat_history")

# 在模板中使用
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手"),
    placeholder,  # 等价于 MessagesPlaceholder("chat_history")
    ("human", "{input}"),
])

# 传入消息列表
messages = prompt.format_messages(
    chat_history=[
        HumanMessage(content="你好"),
        AIMessage(content="你好！有什么可以帮你的？"),
    ],
    input="我想了解Python"
)

for msg in messages:
    print(f"{msg.type}: {msg.content}")
# system: 你是一个助手
# human: 你好
# ai: 你好！有什么可以帮你的？
# human: 我想了解Python
```

## 可选的历史消息

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# 使用 optional=True，如果没有传入则跳过
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手"),
    MessagesPlaceholder("chat_history", optional=True),
    ("human", "{input}"),
][[)

# 不传 chat_history 也不会报错
messages = prompt.format_messages(input="你好")
# 只包含 system 和 human 消息

for msg in messages:
    print(f"{msg.type}: {msg.content}")
# system: 你是一个助手
# human: 你好
```

## 相关页面

- [[langchain/modules/langchain_prompts/概述|prompts 模块概述]]
- [[langchain/modules/langchain_prompts/ChatPromptTemplate|ChatPromptTemplate]] — 通常配合使用
- [[langchain/modules/langchain_prompts/Prompt应用场景|应用场景]] — 多轮对话 Agent 示例
