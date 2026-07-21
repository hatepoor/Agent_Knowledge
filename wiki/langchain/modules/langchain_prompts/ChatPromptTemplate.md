---
tags: [langchain, prompts, ChatPromptTemplate, chat, template]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# ChatPromptTemplate

`ChatPromptTemplate` 是 LangChain 中**最常用的模板类**，用于创建包含多条消息（system、human、ai 等）的聊天提示词。它输出消息列表而非纯字符串，天然适配当前主流的 chat 模型（如 `ChatOpenAI`、`ChatAnthropic`）。

## 基础用法

```python
from langchain_core.prompts import ChatPromptTemplate

# 方式1：from_messages（推荐）
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{domain}专家"),
    ("human", "请解释{concept}"),
])

# 格式化为消息列表
messages = prompt.format_messages(
    domain="物理学",
    concept="相对论"
)

for msg in messages:
    print(f"{msg.type}: {msg.content}")
# system: 你是一个物理学专家
# human: 请解释相对论

# 方式2：from_template（单条消息）
prompt = ChatPromptTemplate.from_template("请解释{concept}")
# 等价于 [("human", "请解释{concept}")]
```

## 消息类型

```python
from langchain_core.prompts import ChatPromptTemplate

# 支持的消息类型
prompt = ChatPromptTemplate.from_messages([
    ("system", "系统消息"),      # SystemMessage
    ("human", "用户消息"),       # HumanMessage
    ("ai", "AI 消息"),          # AIMessage
    ("user", "用户消息"),       # 等价于 human
    ("assistant", "助手消息"),  # 等价于 ai
])

# 也可以使用元组格式
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手"),
    ("human", "{question}"),
])
```

### 方法返回类型

| 方法 | 返回类型 | 说明 |
|------|---------|------|
| `ChatPromptTemplate.from_messages(...)` | `ChatPromptTemplate` | 从消息列表创建模板 |
| `ChatPromptTemplate.from_template(...)` | `ChatPromptTemplate` | 从单条消息创建模板 |
| `.format_messages(...)` | `list[BaseMessage]` | 格式化为消息列表 |
| `.format(...)` | `str` | 格式化为纯字符串 |
| `.invoke(...)` | `ChatPromptValue` | Runnable 接口调用 |
| `.partial(...)` | `ChatPromptTemplate` | 预填充部分变量 |
| `.to_string()` | `str` | PromptValue 转字符串 |
| `.to_messages()` | `list[BaseMessage]` | PromptValue 转消息列表 |

### 类型验证示例

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}"),
    ("human", "{question}"),
])
print(type(prompt))  # <class 'ChatPromptTemplate'>

messages = prompt.format_messages(role="专家", question="什么是AI？")
print(type(messages))  # <class 'list'>
print(type(messages[0]))  # <class 'SystemMessage'>

prompt_value = prompt.invoke({"role": "专家", "question": "什么是AI？"})
print(type(prompt_value))  # <class 'ChatPromptValue'>
```

## 带历史消息的模板

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage

# 使用 MessagesPlaceholder 插入历史
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手"),
    MessagesPlaceholder("chat_history"),  # 历史消息占位符
    ("human", "{input}"),
])

# 调用时传入历史
messages = prompt.format_messages(
    chat_history=[
        HumanMessage(content="什么是Python？"),
        AIMessage(content="Python是一种编程语言"),
    ],
    input="它有什么特点？"
)

for msg in messages:
    print(f"{msg.type}: {msg.content}")
# system: 你是一个有帮助的助手
# human: 什么是Python？
# ai: Python是一种编程语言
# human: 它有什么特点？
```

## 部分变量预填充

```python
from langchain_core.prompts import ChatPromptTemplate
from datetime import datetime

# 预填充某些变量
prompt = ChatPromptTemplate.from_messages([
    ("system", "当前日期：{date}。你是一个{role}"),
    ("human", "{question}"),
]).partial(
    date=datetime.now().strftime("%Y-%m-%d"),
    role="技术顾问"
)

# 调用时只需传 question
messages = prompt.format_messages(question="有什么建议？")

for msg in messages:
    print(f"{msg.type}: {msg.content}")
# system: 当前日期：2026-07-08。你是一个技术顾问
# human: 有什么建议？
```

## 多轮对话模板

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage

# 创建多轮对话模板
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}。请用{language}回答"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

# 模拟多轮对话
chat_history = []

# 第一轮
messages = prompt.format_messages(
    role="Python 专家",
    language="中文",
    chat_history=chat_history,
    input="什么是装饰器？"
)
# 调用 LLM...
# 假设得到回答
chat_history.extend([
    HumanMessage(content="什么是装饰器？"),
    AIMessage(content="装饰器是一种函数..."),
][[)

# 第二轮
messages = prompt.format_messages(
    role="Python 专家",
    language="中文",
    chat_history=chat_history,
    input="能给个例子吗？"
)
# 这次 chat_history 包含了第一轮的历史
```

## 相关页面

- [[langchain/modules/langchain_prompts/概述|prompts 模块概述]]
- [[langchain/modules/langchain_prompts/from_template方法|from_template 方法]]
- [[langchain/modules/langchain_prompts/MessagesPlaceholder|MessagesPlaceholder]] — 历史消息占位符
- [[langchain/modules/langchain_prompts/PromptTemplate|PromptTemplate]] — 基础字符串模板
