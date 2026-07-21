---
tags: [langchain, agent, create_agent, parameters]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# create_agent 完整参数

## 概述

`create_agent` 是 LangChain v1 中创建 Agent 的唯一工厂函数，支持丰富的配置选项。本页列出所有可用参数及其详细说明。

## 参数表

| 参数 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `model` | str \| BaseChatModel | ✅ | 模型标识符或实例 |
| `tools` | list[BaseTool \| Callable \| dict] | ✅ | 工具列表 |
| `system_prompt` | str \| SystemMessage | ❌ | 角色定义 |
| `state_schema` | type[AgentState] | ❌ | 自定义状态 |
| `context_schema` | type[Context] | ❌ | 运行时上下文 |
| `checkpointer` | BaseCheckpointSaver | ❌ | 短期记忆后端 |
| `store` | BaseStore | ❌ | 长期记忆后端 |
| `middleware` | list[AgentMiddleware][[ | ❌ | 中间件列表 |
| `response_format` | type \| Strategy | ❌ | 结构化输出 |
| `name` | str | ❌ | Agent 标识（多 Agent 场景） |

## 参数详解

### model（必填）

接受两种形式：
- **字符串标识符**：`"openai:gpt-5.5"`、`"anthropic:claude-sonnet-4-6"` 等，格式为 `provider:model_name`
- **模型实例**：直接传入 `ChatOpenAI`、`ChatAnthropic` 等已初始化的模型对象

### tools（必填）

工具列表，支持三种形式：
- `@tool` 装饰的函数（自动转为 `BaseTool`）
- 继承 `BaseTool` 的类实例
- `dict` 形式的工具定义

> 所有工具函数必须有 **docstring**，该描述会传给模型用于决策。

### system_prompt

系统提示词，定义 Agent 的角色和行为边界。可以是纯字符串或 `SystemMessage` 对象。

### state_schema

自定义 Agent 状态类型，继承自 `AgentState`。可用于追踪自定义字段（如计数器、标志位等）。

```python
class AppState(AgentState):
    request_count: int
    last_query: str
```

### context_schema

运行时上下文类型（通常为 `@dataclass`），在每次 `invoke` 时传入，不会持久化到 state 中。

```python
@dataclass
class AppContext:
    user_id: str
    session_id: str
```

### checkpointer

短期记忆后端，用于按 `thread_id` 持久化对话历史。常用 `InMemorySaver`（开发）或数据库实现（生产）。

### store

长期记忆后端，用于跨会话存储。工具中通过 `ToolRuntime.store` 访问。

### middleware

中间件列表，可在 Agent 执行前后插入逻辑：
- 上下文管理
- 容错与重试
- 护栏（Guardrails）
- 人工审批（Human-in-the-loop）

### response_format

结构化输出配置，确保 Agent 返回指定格式的数据而非自由文本。

### name

Agent 标识名，在多 Agent 协作场景中用于区分不同 Agent。

## 相关页面

- [[langchain/modules/langchain_agents/概述|← 返回 agents 概述]]
- [[langchain/modules/langchain_agents/Agent与LangGraph关系|Agent 与 LangGraph 的关系]]
