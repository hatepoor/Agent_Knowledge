---
tags: [langchain, agents, core, source]
created: 2026-07-07
updated: 2026-07-07
type: source
source_url: https://docs.langchain.com/oss/python/langchain/agents
---

# Agent（代理）

Agent 是一个模型在循环中调用工具直到给定任务完成的过程。

**Agent = 模型 + 框架（Harness）**

框架的职责：在正确的时间为模型提供正确的上下文来完成给定任务。框架涵盖了该循环的一切：模型、其提示词、其工具以及任何塑造其行为的中间件。

`create_agent` 是一个高度可配置的框架。最简单的用法：

```python
from langchain.agents import create_agent

agent = create_agent(model="google_genai:gemini-3.5-flash", tools=tools)
```

## 核心组件

### 模型

传递模型标识符字符串（`"provider:model"`）或已初始化的模型实例：

```python
agent = create_agent(model="google_genai:gemini-3.5-flash", tools=tools)
```

### 工具

传递任何 Python 可调用对象、LangChain 工具或工具字典：

```python
from langchain.tools import tool

@tool
def search(query: str) -> str:
    """Search for information."""
    return f"Results for: {query}"

agent = create_agent(model="google_genai:gemini-3.5-flash", tools=[search])
```

### 系统提示词

塑造 Agent 处理任务的方式，接受字符串或 `SystemMessage`：

```python
agent = create_agent(
    model="google_genai:gemini-3.5-flash",
    tools=tools,
    system_prompt="You are a helpful assistant. Be concise and accurate.",
)
```

### 结构化输出

使用 `response_format=` 从 Agent 返回经过验证的模式：

```python
from pydantic import BaseModel

class Answer(BaseModel):
    summary: str
    confidence: float

agent = create_agent(model="google_genai:gemini-3.5-flash", tools=tools, response_format=Answer)
result = agent.invoke({"messages": [{"role": "user", "content": "Summarize AI trends"}]})
result["structured_response"]  # Answer(summary=..., confidence=...)
```

## 调用

你可以通过消息调用 Agent。所有 Agent 在其状态中包含[消息序列](https://docs.langchain.com/oss/python/langgraph/graph-api#state)。要调用 Agent，请传递新消息以及 `thread_id` 以便 Agent 能持久化并恢复对话历史：

```python
from langchain.agents import create_agent
from langchain_core.utils.uuid import uuid7
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="google_genai:gemini-3.5-flash",
    tools=[],
    checkpointer=InMemorySaver(),
)

config = {"configurable": {"thread_id": str(uuid7())}}

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]},
    config=config,
)

# 同一对话的后续轮次：重用相同的 thread_id 以保留历史记录
result = agent.invoke(
    {"messages": [{"role": "user", "content": "What about tomorrow?"}]},
    config=config,
)
```

通过 `thread_id` 持久化对话历史要求 Agent 配置了 [checkpointer](https://docs.langchain.com/oss/python/langchain/long-term-memory)。

### 传递每次运行的上下文

如果需要向工具和中间件传递每次运行的配置（如用户 ID、API 密钥或功能标志），将其作为 `context` 与 `config` 一起传递。通过 `context_schema` 定义数据形状，通过 `runtime.context` 访问：

```python
from dataclasses import dataclass

@dataclass
class Context:
    user_id: str

agent = create_agent(
    model="google_genai:gemini-3.5-flash",
    tools=[],
    context_schema=Context,
    checkpointer=InMemorySaver(),
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather?"}]},
    config={"configurable": {"thread_id": str(uuid7())}},
    context=Context(user_id="user-123"),
)
```

## 流式传输

`invoke` 在运行结束时返回最终响应。如果 Agent 执行多个工具调用，用户通常需要进度更新。使用流式传输在发生时就呈现中间消息和工具活动：

```python
from langchain.messages import AIMessage, HumanMessage

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "Search for AI news and summarize"}]},
    version="v3",
)
for snapshot in stream.values:
    latest_message = snapshot["messages"][-1]
    if latest_message.content:
        if isinstance(latest_message, HumanMessage):
            print(f"User: {latest_message.content}")
        elif isinstance(latest_message, AIMessage):
            print(f"Agent: {latest_message.content}")
    elif latest_message.tool_calls:
        print(f"Calling tools: {[tc['name'] for tc in latest_message.tool_calls]}")
```

## 配置框架（Harness）

`create_agent` 高度可扩展。中间件是自定义的基础组件：每个中间件处理一个关注点，在正确时机挂入 Agent 循环，并可自由地与任何其他中间件组合。

### 执行环境

```python
from deepagents.backends import StateBackend
from deepagents.middleware import FilesystemMiddleware

agent = create_agent(
    model="google_genai:gemini-3.5-flash",
    tools=[search],
    middleware=[FilesystemMiddleware(backend=StateBackend())],
)
```

### 上下文管理

每次模型调用都有固定的上下文窗口。随着 Agent 运行，该窗口会填满累积的历史记录、工具结果和中间步骤。摘要化（Summarization）在溢出前压缩历史；记忆（Memory）在启动时加载持久指令；技能（Skills）按需呈现领域知识：

```python
from deepagents.middleware import (
    FilesystemMiddleware, MemoryMiddleware, SkillsMiddleware, SummarizationMiddleware
)

agent = create_agent(
    model=model,
    tools=[search],
    middleware=[
        FilesystemMiddleware(backend=backend),
        SummarizationMiddleware(model=model, backend=backend),
        MemoryMiddleware(backend=backend, sources=["./AGENTS.md"]),
        SkillsMiddleware(backend=backend, sources=["./skills/"]),
    ],
)
```

### 规划与委托

复杂任务通常超出单个上下文窗口的处理能力。委托让主 Agent 将工作拆分为碎片，交给子 Agent 处理，保持主 Agent 专注于协调：

```python
from deepagents.middleware.subagents import SubAgentMiddleware
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(
    model="google_genai:gemini-3.5-flash",
    tools=[search],
    middleware=[
        FilesystemMiddleware(backend=backend),
        TodoListMiddleware(),
        SubAgentMiddleware(
            backend=backend,
            subagents=[
                {
                    "name": "researcher",
                    "description": "Searches and returns a structured summary.",
                    "system_prompt": "Use the search tool to research...",
                    "tools": [search],
                    "model": "anthropic:claude-sonnet-4-6",
                }
            ],
        ),
    ],
)
```

### 容错

```python
from langchain.agents.middleware import ModelRetryMiddleware, ToolRetryMiddleware

agent = create_agent(
    model="google_genai:gemini-3.5-flash",
    tools=[search],
    middleware=[
        ModelRetryMiddleware(max_retries=3),
        ToolRetryMiddleware(max_retries=2),
    ],
)
```

### 护栏

```python
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="google_genai:gemini-3.5-flash",
    tools=[search],
    middleware=[PIIMiddleware("email")],
)
```

### 人机交互

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware

agent = create_agent(
    model="google_genai:gemini-3.5-flash",
    tools=[search],
    middleware=[HumanInTheLoopMiddleware(interrupt_on={"write_file": True})][[,
)
```

## 相关页面

- [[langchain/agents/Agent循环实现]] — ReAct 循环四层架构与内部实现原理
- [[langchain/tools/工具]] — 工具系统详解
- [[langchain/models/模型]] — 模型配置
- [[langchain/streaming/流式传输]] — 流式传输
- [[langchain/middleware/中间件概述]] — 中间件生态
