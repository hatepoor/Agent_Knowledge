---
tags: [langchain, output-parsers, string]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# StrOutputParser

## 是什么

`StrOutputParser` 是最简单也最常用的输出解析器。它从 LLM 返回的 `AIMessage` 对象中提取 `.content` 属性，返回一个纯字符串。

## 导入

```python
from langchain_core.output_parsers import StrOutputParser
```

## 内部原理

`StrOutputParser` 的核心逻辑极其简单——本质上就是读取 `AIMessage.content`：

```python
from langchain_core.output_parsers import StrOutputParser

parser = StrOutputParser()

# 内部等价于：
# result = ai_message.content

# 手动演示：
from langchain_core.messages import AIMessage

msg = AIMessage(content="你好世界")
print(parser.parse(msg))  # "你好世界"
```

它不会修改、清洗或转换内容，只是原样提取 `content` 字段。

## 基础用法

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手。"),
    ("human", "讲一个关于{topic}的笑话。"),
])

chain = prompt | model | StrOutputParser()
result = chain.invoke({"topic": "编程"})
print(result)  # 纯文本字符串
print(type(result))  # <class 'str'>
```

## tool_call 消息的特殊行为

当模型返回的是 **tool_call 响应**（即模型请求调用工具而非直接回复文本）时，`AIMessage.content` 通常是 **空字符串 `""`**，而非 `None`：

```python
from langchain_core.messages import AIMessage

# 模拟一个 tool_call 响应
msg = AIMessage(content="", additional_kwargs={
    "tool_calls": [{
        "id": "call_abc123",
        "function": {"name": "get_weather", "arguments": '{"city": "北京"}'},
        "type": "function"
    }]
})

parser = StrOutputParser()
result = parser.parse(msg)
print(repr(result))  # ''（空字符串，不是 None）
print(len(result))   # 0
```

这意味着：

- `StrOutputParser` **不会抛出异常**，它会静默返回空字符串
- 如果你的链可能触发 tool_call，需要额外检查返回值是否为空
- 在使用 bind_tools 的链中，建议使用 `JsonOutputParser` 或直接访问 `AIMessage.tool_calls`

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

# 如果这个链可能返回 tool_call，需要额外处理
chain = prompt | model | StrOutputParser()
result = chain.invoke({"topic": "天气"})

if result == "":
    print("警告：模型可能返回了 tool_call 而非文本内容")
```

## 异步用法

`StrOutputParser` 支持异步调用，适合在 async 框架（FastAPI、aiohttp 等）中使用：

### ainvoke - 异步单次调用

```python
import asyncio
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手。"),
    ("human", "用一句话介绍{city}。"),
])

chain = prompt | model | StrOutputParser()

async def main():
    result = await chain.ainvoke({"city": "北京"})
    print(result)

asyncio.run(main())
```

### astream - 异步流式输出

```python
import asyncio
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手。"),
    ("human", "写一首关于{topic}的短诗。"),
])

chain = prompt | model | StrOutputParser()

async def main():
    async for chunk in chain.astream({"topic": "秋天"}):
        print(chunk, end="", flush=True)

asyncio.run(main())
```

### abatch - 异步批量调用

```python
import asyncio
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_messages([
    ("human", "用一句话介绍{city}。"),
])

chain = prompt | model | StrOutputParser()

async def main():
    results = await chain.abatch([
        {"city": "北京"},
        {"city": "上海"},
        {"city": "广州"},
    ])
    for city, result in zip(["北京", "上海", "广州"], results):
        print(f"{city}: {result}")

asyncio.run(main())
```

## 错误边界情况

### 空响应

某些情况下模型可能返回完全空的 content：

```python
from langchain_core.messages import AIMessage
from langchain_core.output_parsers import StrOutputParser

parser = StrOutputParser()

# 空内容 — 不会报错，返回空字符串
msg = AIMessage(content="")
result = parser.parse(msg)
print(repr(result))  # ''
```

### None content

极少数情况下，`content` 可能是 `None`（例如某些非标准模型适配器）：

```python
from langchain_core.messages import AIMessage
from langchain_core.output_parsers import StrOutputParser

parser = StrOutputParser()

# content 为 None 时会抛出异常
msg = AIMessage(content=None)
try:
    result = parser.parse(msg)
except Exception as e:
    print(f"解析错误: {e}")  # 可能抛出 AttributeError 或 TypeError
```

### 防御性编程建议

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
chain = ChatPromptTemplate.from_messages([
    ("human", "{question}"),
]) | model | StrOutputParser()

result = chain.invoke({"question": "什么是Python？"})

# 防御性检查
if not result:
    print("收到空响应，可能需要重试")
elif len(result) < 10:
    print(f"响应异常短: {result}")
else:
    print(result)
```

## 实用技巧：何时不需要 StrOutputParser

如果你只是手动访问 `.content`，可以跳过 `StrOutputParser`：

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_messages([
    ("human", "你好"),
])

# 方式1：使用 StrOutputParser（推荐用于 LCEL 链）
chain = prompt | model | StrOutputParser()
text = chain.invoke({})

# 方式2：直接访问 .content（不需要 StrOutputParser）
chain_without_parser = prompt | model
ai_message = chain_without_parser.invoke({})
text = ai_message.content  # 效果完全相同
```

**何时使用 `StrOutputParser`：**

- 在 LCEL 链中需要 `str` 类型输出时
- 需要与其他解析器保持接口一致时
- 需要将链的输出直接传给只接受字符串的下游函数时

**何时不需要：**

- 你已经在手动处理 `AIMessage` 对象
- 你需要访问 `.response_metadata`、`.usage_metadata` 等额外信息
- 链的下游需要 `AIMessage` 而非 `str`

## 流式输出

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手。"),
    ("human", "讲一个关于{topic}的故事。"),
][[)

chain = prompt | model | StrOutputParser()

for chunk in chain.stream({"topic": "小猫"}):
    print(chunk, end="", flush=True)
```

## 相关

- [[langchain/modules/langchain_output_parsers/概述|← 返回 output_parsers 概述]]
- [[langchain/modules/langchain_output_parsers/ListOutputParser|ListOutputParser - 列表解析]]
- [[langchain/modules/langchain_output_parsers/JsonOutputParser|JsonOutputParser - JSON 解析]]
- [[langchain/modules/langchain_output_parsers/with_structured_output|with_structured_output - 现代替代方案]]
- [[langchain/modules/langchain_output_parsers/错误处理|错误处理]]
