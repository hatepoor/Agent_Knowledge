---
tags: [concept, LangChain, Agent, ChatModel, invoke, 方法对比]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# Agent invoke vs ChatModel invoke

> `create_agent().invoke()` 和 `ChatModel.invoke()` 都实现了 Runnable 接口，但工作在完全不同的抽象层级上，返回内容的结构和含义有本质区别。

## 核心区别一句话

- **ChatModel.invoke()** → 单次 LLM API 调用，返回一个 `AIMessage`
- **Agent.invoke()** → 完整 Agent 循环（可能多次 LLM 调用 + 工具执行），返回一个状态 `dict`

## 返回类型对比

| 维度 | ChatModel.invoke() | Agent.invoke() |
|------|-------------------|----------------|
| 返回类型 | `AIMessage` 对象 | `dict`（状态字典） |
| 最终回答 | `response.content` 直接获取 | `result["messages"][-1].content` |
| 过程记录 | 无，只有一次调用 | `result["messages"]` 包含所有中间步骤 |
| 工具调用 | `tool_calls` 只有意图，无执行结果 | 既有 AIMessage 的调用意图，也有 ToolMessage 的执行结果 |
| 自定义字段 | 无 | 有（如果定义了 state_schema） |
| LLM 调用次数 | 1 次 | 可能多次（循环直到模型不再调用工具） |
| 状态管理 | 无状态 | 有状态（AgentState + checkpointer） |
| 多轮对话 | 需手动拼接消息历史 | checkpointer 自动管理 |
| 适用场景 | 简单问答、单次生成、结构化输出 | 需要工具协作的多步推理任务 |

## ChatModel.invoke() 返回结构

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")
response = llm.invoke("你好")

# response 是 AIMessage 对象
type(response)  # langchain_core.messages.ai.AIMessage

# 核心属性
response.content           # "你好！有什么可以帮你的？"
response.tool_calls        # []（没绑工具时为空）
response.response_metadata # {"model": "gpt-4o-mini", "finish_reason": "stop", ...}
response.id                # "chatcmpl-xxx"
response.usage_metadata    # {"input_tokens": 10, "output_tokens": 25, ...}
```

### 绑定工具后的 ChatModel

即使通过 `bind_tools` 绑定了工具，`invoke` 也**不会执行工具**，只返回调用意图：

```python
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get weather for a city."""
    return f"Sunny in {city}"

llm_with_tools = ChatOpenAI(model="gpt-4o-mini").bind_tools([get_weather])
response = llm_with_tools.invoke("东京天气怎么样？")

# response 仍然是 AIMessage
response.content     # ""（模型选择调用工具时，content 通常为空）
response.tool_calls  # [{"name": "get_weather", "args": {"city": "东京"}, "id": "call_xxx"}]

# 注意：工具并没有被执行！你需要自己：
# 1. 解析 tool_calls
# 2. 调用对应工具
# 3. 将结果作为 ToolMessage 拼回消息列表
# 4. 再次调用 LLM 获取最终回答
```

## Agent.invoke() 返回结构

```python
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get weather for a city."""
    return f"Sunny in {city}, 28°C"

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[get_weather],
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "东京天气怎么样？"}]}
)

# result 是 dict
type(result)  # dict

# 核心：messages 列表包含完整执行过程
result["messages"]
# [
#   HumanMessage("东京天气怎么样？"),
#   AIMessage(tool_calls=[{"name": "get_weather", "args": {"city": "东京"}}]),
#   ToolMessage("Sunny in Tokyo, 28°C", tool_call_id="call_xxx"),
#   AIMessage("东京今天天气晴朗，气温28°C。"),
# ]

# 获取最终回答
final_answer = result["messages"][-1].content
# "东京今天天气晴朗，气温28°C。"
```

### 每条消息的含义

```python
for i, msg in enumerate(result["messages"]):
    print(f"[{i}] {type(msg).__name__}: {msg.content[:50] if msg.content else msg.tool_calls}")

# [0] HumanMessage: 东京天气怎么样？
# [1] AIMessage: [{'name': 'get_weather', 'args': {'city': '东京'}}]  ← 模型决定调用工具
# [2] ToolMessage: Sunny in Tokyo, 28°C                              ← 工具执行结果
# [3] AIMessage: 东京今天天气晴朗，气温28°C。                          ← 最终回答
```

## 自定义状态对返回内容的影响

当使用 `state_schema` 扩展了默认 `AgentState` 时，返回的 dict 会包含自定义字段：

```python
from langchain.agents import create_agent, AgentState

class AppState(AgentState):
    request_count: int
    last_tool_used: str

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[get_weather],
    state_schema=AppState,
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "东京天气？"}]},
    config={"configurable": {"thread_id": "user-1"}},
)

# 除了 messages，还有自定义字段
result["messages"]       # 完整消息列表
result["request_count"]  # 自定义字段的值
result["last_tool_used"] # 自定义字段的值
```

## 实际使用对比

### ChatModel：需要手动编写工具调用循环

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini").bind_tools([get_weather])
messages = [{"role": "user", "content": "东京天气？"}]

# 手动循环
while True:
    response = llm.invoke(messages)  # 每次只调一次 LLM
    
    if not response.tool_calls:
        break  # 模型不再调用工具，结束
    
    # 手动执行每个工具
    for tc in response.tool_calls:
        result = execute_tool(tc["name"], tc["args"])  # 需自己实现
        messages.append(response)  # AI 的调用意图
        messages.append(ToolMessage(content=result, tool_call_id=tc["id"]))

# 最终回答在 response.content 中
print(response.content)
```

### Agent：一行代码搞定

```python
from langchain.agents import create_agent

agent = create_agent(model="openai:gpt-4o-mini", tools=[get_weather])

result = agent.invoke(
    {"messages": [{"role": "user", "content": "东京天气？"}]}
)

# 所有工具调用、结果收集、最终回答都在一次 invoke 中完成
print(result["messages"][-1].content)
```

## 选型指南

| 场景 | 推荐 | 原因 |
|------|------|------|
| 简单问答（不需要工具） | ChatModel.invoke() | 开销最小，一次 API 调用 |
| 结构化输出 | ChatModel + with_structured_output | 模型直接返回 Pydantic 对象 |
| 单次工具调用 + 自己处理结果 | ChatModel + bind_tools | 灵活控制工具执行逻辑 |
| 多步工具调用 | Agent.invoke() | 自动循环，自动收集结果 |
| 需要对话记忆 | Agent + checkpointer | 自动管理消息历史 |
| 需要追踪中间步骤 | Agent.invoke() | messages 列表包含完整日志 |
| 流式输出 | 两者都支持 | ChatModel 用 stream()，Agent 用 stream() |

## 常见陷阱

### 陷阱 1：期望 ChatModel.invoke() 自动执行工具

```python
# ❌ 错误认知
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools([get_weather])
response = llm.invoke("东京天气？")
# 以为工具会被自动执行——不会！
# response.tool_calls 只有意图，没有结果

# ✅ 正确做法：用 create_agent
agent = create_agent(model="openai:gpt-4o-mini", tools=[get_weather])
result = agent.invoke({"messages": [{"role": "user", "content": "东京天气？"}]})
```

### 陷阱 2：从 Agent 返回中直接取 content

```python
result = agent.invoke({"messages": [{"role": "user", "content": "东京天气？"}]})

# ❌ 错误：result 是 dict，没有 .content 属性
print(result.content)  # AttributeError!

# ✅ 正确：从 messages 最后一条取
print(result["messages"][-1].content)
```

### 陷阱 3：忽略中间消息导致调试困难

```python
# ❌ 只取最终回答，丢失调试信息
result = agent.invoke({"messages": [...]})
print(result["messages"][-1].content)

# ✅ 调试时打印所有消息，看清 Agent 做了什么
for msg in result["messages"][[:
    print(f"{type(msg).__name__}: {msg.content or msg.tool_calls}")
```

## 相关页面

- [[langchain/basics/LangChain核心方法对比]] — invoke/stream/embed 方法的基础对比
- [[langchain/modules/langchain_agents/概述]] — create_agent 完整参数与用法
- [[langchain/modules/langchain_models/概述]] — ChatModel 统一接口详解
- [[langchain/agents/Agent代理]] — Agent 核心概念：模型 + 框架（Harness）
- [[langchain/agents/Agent循环实现]] — ReAct 循环四层架构详解
