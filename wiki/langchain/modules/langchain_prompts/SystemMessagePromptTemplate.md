---
tags: [langchain, prompts, SystemMessagePromptTemplate, system]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# SystemMessagePromptTemplate

`SystemMessagePromptTemplate` 专门用于创建**系统消息**（SystemMessage）。系统消息用于设定 AI 的行为规则、角色定位和输出格式，在聊天提示词中通常放在最前面。

> 大多数情况下可以直接在 `ChatPromptTemplate.from_messages()` 中使用 `("system", "...")` 元组，效果相同。`SystemMessagePromptTemplate` 的优势在于可以独立创建和复用系统消息模板。

## 基础用法

```python
from langchain_core.prompts import (
    ChatPromptTemplate,
    SystemMessagePromptTemplate,
    HumanMessagePromptTemplate,
)

# 方式1：直接在 from_messages 中使用（简洁）
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}"),
    ("human", "{question}"),
])

# 方式2：使用 SystemMessagePromptTemplate（更灵活，可独立复用）
system_template = SystemMessagePromptTemplate.from_template(
    "你是一个{role}。请用{language}回答"
)
human_template = HumanMessagePromptTemplate.from_template(
    "问题：{question}"
)

prompt = ChatPromptTemplate.from_messages([
    system_template,
    human_template,
][[)

messages = prompt.format_messages(
    role="Python 专家",
    language="中文",
    question="什么是装饰器？"
)

for msg in messages:
    print(f"{msg.type}: {msg.content}")
# system: 你是一个Python 专家。请用中文回答
# human: 问题：什么是装饰器？
```

## 相关页面

- [[langchain/modules/langchain_prompts/概述|prompts 模块概述]]
- [[langchain/modules/langchain_prompts/ChatPromptTemplate|ChatPromptTemplate]] — 聊天模板
- [[langchain/modules/langchain_prompts/from_template方法|from_template 方法]]
