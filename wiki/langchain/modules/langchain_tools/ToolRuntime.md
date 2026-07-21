---
tags: [langchain, tools, ToolRuntime, runtime]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# ToolRuntime 运行时参数

## 概述

`ToolRuntime` 是 LangChain 工具的运行时参数对象，通过 `runtime` 参数名注入到工具函数中。它提供了对当前运行环境的访问，包括状态、上下文、长期存储和工具调用 ID 等。

> `ToolRuntime` 取代了旧版的 `InjectedState` 和 `InjectedStore`，推荐统一使用。

## 可访问的属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `runtime.state` | dict | 当前 Agent 状态 |
| `runtime.context` | dataclass | 运行时上下文（invoke 时传入） |
| `runtime.store` | BaseStore | 长期记忆存储 |
| `runtime.tool_call_id` | str | 当前工具调用的 ID |

## 使用方式

在工具函数签名中添加 `runtime: ToolRuntime` 参数即可，系统会自动注入：

```python
from langchain.tools import tool, ToolRuntime

@tool
def get_user_info(runtime: ToolRuntime) -> str:
    """Get current user info from context."""
    user_id = runtime.context.user_id
    return f"User: {user_id}"
```

## 访问长期存储（Store）

```python
@tool
def save_preference(key: str, value: str, runtime: ToolRuntime) -> str:
    """Save user preference to long-term store."""
    runtime.store.put(("prefs", runtime.context.user_id), key, value)
    return f"Saved {key}={value}"

@tool
def read_preference(key: str, runtime: ToolRuntime) -> str:
    """Read user preference from long-term store."""
    item = runtime.store.get(("prefs", runtime.context.user_id), key)
    return item.value if item else "Not set"
```

## 与 Agent 配合使用

```python
from dataclasses import dataclass
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore

@dataclass
class Context:
    user_id: str

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_user_info, save_preference, read_preference],
    context_schema=Context,
    checkpointer=InMemorySaver(),
    store=InMemoryStore(),
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "Save my favorite color blue"}][[},
    config={"configurable": {"thread_id": "thread-1"}},
    context=Context(user_id="user-1"),
)
```

## 注意事项

- `runtime` 是**保留参数名**，不可用于自定义参数
- `ToolRuntime` 仅在 Agent 上下文中可用，直接调用工具函数时不会自动注入
- 推荐使用 `ToolRuntime` 而非已弃用的 `InjectedState` / `InjectedStore`

## 相关页面

- [[langchain/modules/langchain_tools/概述|← 返回 tools 概述]]
- [[langchain/modules/langchain_tools/工具返回值与状态更新|返回值类型与状态更新]]
- [[langchain/modules/langchain_stores/概述|stores 模块]]
