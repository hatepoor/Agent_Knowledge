---
tags: [langchain, prompts, advanced, tips, pitfalls, best-practices]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# Prompt 高级技巧

本文涵盖 `langchain_core.prompts` 的高级用法、常见陷阱、最佳实践和性能优化建议。

## 高级技巧

### 1. 模板组合

```python
from langchain_core.prompts import ChatPromptTemplate

# 创建子模板
system_template = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}"),
])

human_template = ChatPromptTemplate.from_messages([
    ("human", "{question}"),
])

# 组合模板
combined = ChatPromptTemplate.from_messages(
    system_template.messages + human_template.messages
)

messages = combined.format_messages(role="专家", question="什么是AI？")

for msg in messages:
    print(f"{msg.type}: {msg.content}")
# system: 你是一个专家
# human: 什么是AI？
```

### 2. 条件变量

```python
from langchain_core.prompts import ChatPromptTemplate

# 使用 partial 实现条件变量
base_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手"),
    ("human", "{question}"),
])

# 根据不同场景预填充
formal_prompt = base_prompt.partial(
    question="请以正式的方式回答"
)

casual_prompt = base_prompt.partial(
    question="请用轻松的方式回答"
)
```

### 3. 动态系统提示

```python
from langchain_core.prompts import ChatPromptTemplate
from datetime import datetime

# 使用 partial 注入动态值（lambda 实现每次调用时更新）
prompt = ChatPromptTemplate.from_messages([
    ("system", "当前时间：{time}。你是一个{role}"),
    ("human", "{input}"),
]).partial(
    time=lambda: datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
    role="技术顾问"
)

# 每次调用时 time 会自动更新
messages = prompt.format_messages(input="现在几点？")
```

### 4. 模板验证

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}"),
    ("human", "{question}"),
])

# 检查需要的变量
print(prompt.input_variables)  # ['role', 'question']

# 验证变量是否完整
try:
    messages = prompt.format_messages(role="专家")  # 缺少 question
except Exception as e:
    print(f"错误：{e}")
```

## 常见陷阱

### 1. 变量名不匹配

```python
from langchain_core.prompts import ChatPromptTemplate

# 变量名不一致
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}"),
    ("human", "{question}"),
])

# 传入的变量名必须与模板中的 {变量} 完全匹配
messages = prompt.format_messages(
    role="专家",
    question="什么是AI？"  # 不能用 query 或其他名称
)
```

### 2. 忘记 MessagesPlaceholder

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# 错误：直接传入消息列表
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手"),
    ("human", "{input}"),
])

# 这样会报错，因为模板中没有定义 chat_history
# messages = prompt.format_messages(
#     input="你好",
#     chat_history=[...]  # 错误！
# )

# 正确：使用 MessagesPlaceholder
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手"),
    MessagesPlaceholder("chat_history"),  # 必须定义占位符
    ("human", "{input}"),
])
```

### 3. 部分变量覆盖

```python
from langchain_core.prompts import ChatPromptTemplate

# 注意：partial 会覆盖同名变量
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}"),
    ("human", "{question}"),
]).partial(role="专家")

# 即使传入 role 也会被 partial 的值覆盖
messages = prompt.format_messages(
    role="助手",  # 会被忽略
    question="什么是AI？"
)
# system 消息仍然是"你是一个专家"
```

### 4. 模板只在调用时填充

```python
from langchain_core.prompts import ChatPromptTemplate

# 模板不会立即填充
prompt = ChatPromptTemplate.from_messages([
    ("system", "当前时间：{time}"),
    ("human", "{question}"),
])

# 此时 time 和 question 还没有值
# 只有在 format_messages() 或链式调用时才会填充
messages = prompt.format_messages(
    time="2026-07-08",
    question="你好"
)
```

## 最佳实践

### 1. 使用 from_template 简化代码

```python
from langchain_core.prompts import PromptTemplate

# 推荐：自动推断 input_variables
prompt = PromptTemplate.from_template("你好，{name}")

# 不推荐：手动指定
prompt = PromptTemplate(
    template="你好，{name}",
    input_variables=["name"]
)
```

### 2. 使用 ChatPromptTemplate 处理聊天场景

```python
from langchain_core.prompts import ChatPromptTemplate, PromptTemplate

# 推荐：使用 ChatPromptTemplate
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手"),
    ("human", "{question}"),
])

# 不推荐：使用 PromptTemplate（不支持多消息）
prompt = PromptTemplate.from_template("你是一个助手。{question}")
```

### 3. 预填充固定变量

```python
from langchain_core.prompts import ChatPromptTemplate

# 推荐：使用 partial 预填充固定值
prompt = ChatPromptTemplate.from_messages([
    ("system", "当前日期：{date}。你是一个助手"),
    ("human", "{question}"),
]).partial(date="2026-07-08")

# 调用时只需传 question
messages = prompt.format_messages(question="你好")
```

### 4. 使用 MessagesPlaceholder 管理历史

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# 推荐：使用 MessagesPlaceholder
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

# 不推荐：手动拼接消息
```

### 5. 验证模板变量

```python
from langchain_core.prompts import ChatPromptTemplate

# 推荐：检查 input_variables
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}"),
    ("human", "{question}"),
])

print(prompt.input_variables)  # ['role', 'question']
# 确保调用时传入所有必需的变量
```

## 性能考虑

### 模板创建 vs 格式化

```python
import time
from langchain_core.prompts import ChatPromptTemplate

# 模板创建（一次性成本）
start = time.time()
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}"),
    ("human", "{question}"),
])
print(f"创建耗时：{time.time() - start:.6f}s")  # ~0.0001s

# 格式化（每次调用）
start = time.time()
for _ in range(1000):
    messages = prompt.format_messages(role="专家", question="你好")
print(f"1000次格式化耗时：{time.time() - start:.4f}s")  # ~0.1s
```

### 缓存模板

```python
from langchain_core.prompts import ChatPromptTemplate

# 推荐：复用模板对象
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手"),
    ("human", "{question}"),
])

# 多次调用
questions = ["什么是AI？", "什么是ML？", "什么是DL？"]
for question in questions:
    messages = prompt.format_messages(question=question)
    # ...

# 不推荐：每次都创建新模板
for question in questions:
    prompt = ChatPromptTemplate.from_messages([
        ("system", "你是一个助手"),
        ("human", "{question}"),
    ][[)  # 浪费
    messages = prompt.format_messages(question=question)
```

## 相关页面

- [[langchain/modules/langchain_prompts/概述|prompts 模块概述]]
- [[langchain/modules/langchain_prompts/ChatPromptTemplate|ChatPromptTemplate]]
- [[langchain/modules/langchain_prompts/MessagesPlaceholder|MessagesPlaceholder]]
- [[langchain/modules/langchain_prompts/Prompt应用场景|应用场景]]
