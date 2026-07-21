---
tags: [langchain, chat-models, bind_tools, tool-calling]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# bind_tools 工具绑定

## 概述

`bind_tools` 是 `BaseChatModel` 提供的方法，用于将工具绑定到模型实例。绑定后，模型在推理时可以选择调用这些工具。这是实现 Agent 工具调用的底层机制。

> `bind_tools` 返回一个新的模型实例（原实例不受影响），符合 LangChain 的不可变设计原则。

## 基本用法

```python
from langchain.chat_models import init_chat_model
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get weather at a location."""
    return f"Sunny in {city}"

# 加载模型
model = init_chat_model("openai:gpt-5.5")

# 绑定工具，返回新实例
model_with_tools = model.bind_tools([get_weather])

# 调用 — 模型可能决定调用工具
response = model_with_tools.invoke("Weather in Tokyo?")

# 检查模型是否请求了工具调用
for tc in response.tool_calls:
    print(f"Tool: {tc['name']}, Args: {tc['args']}")
```

## tool_calls 结构

模型响应中的 `tool_calls` 是一个列表，每个元素包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | str | 工具调用 ID（用于匹配 ToolMessage） |
| `name` | str | 被调用的工具名称 |
| `args` | dict | 传给工具的参数 |
| `type` | str | 固定为 `"tool_call"` |

## 绑定多个工具

```python
@tool
def search(query: str) -> str:
    """Search the web."""
    return f"Results for: {query}"

@tool
def calculate(expression: str) -> str:
    """Calculate a math expression."""
    return str(eval(expression))

model_with_many_tools = model.bind_tools([get_weather, search, calculate])
```

## 与 Agent 的关系

在 Agent 中，`create_agent` 内部自动调用 `bind_tools`，你不需要手动绑定：

```python
# Agent 自动处理工具绑定
agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_weather, search][[,  # 自动 bind_tools
)
```

但在不使用 Agent、直接调用模型时，需要手动 `bind_tools`。

## 相关页面

- [[langchain/modules/langchain_models/概述|← 返回 chat_models 概述]]
- [[langchain/modules/langchain_models/结构化输出|结构化输出]]
- [[langchain/modules/langchain_tools/概述|tools 模块]]
- [[langchain/modules/langchain_agents/概述|agents 模块]]
