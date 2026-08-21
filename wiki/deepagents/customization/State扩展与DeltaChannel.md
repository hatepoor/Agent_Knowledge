---
tags: [deepagents, customization, state, middleware]
created: 2026-07-29
updated: 2026-07-29
type: concept
deepagents_version: ">=0.7"
---

# State 扩展与 DeltaChannel

DeepAgents 默认使用 `DeepAgentState`，其 messages 字段用 **DeltaChannel** reducer——比普通 `add_messages` 省一半 checkpoint 空间。

## 一、DeltaChannel vs add_messages

| | `add_messages` | `DeltaChannel`（DeepAgents 默认） |
|---|---|---|
| 每步存储 | 完整消息列表 | 只存增量 |
| checkpoint 大小增长 | O(N²) | **O(N)** |
| 效果 | 100 步 → ~50MB | 100 步 → ~25MB |

```python
# 源码（deepagents/__init__.py）
class DeepAgentState(AgentState):
    messages: Annotated[list[BaseMessage], DeltaChannel]
    # ↑ DeltaChannel 是 add_messages 的优化版本
    #   每次 checkpoint 只保存新增的消息，不重复存储旧消息
```

**零配置，自动生效**。

## 二、扩展 State

给 Agent 添加自定义状态字段（中间件可读写）：

```python
from typing_extensions import NotRequired
from deepagents import DeepAgentState

class MyAgentState(DeepAgentState):
    """扩展 state：追踪自定义指标"""
    model_call_count: NotRequired[int]
    tool_call_count: NotRequired[int]
    user_tier: NotRequired[str]       # free | pro | enterprise
    last_error: NotRequired[str]

# 中间件中读写
from langchain.agents.middleware import AgentMiddleware

class CountingMiddleware(AgentMiddleware):
    state_schema = MyAgentState

    def after_model(self, state: MyAgentState, runtime):
        count = state.get("model_call_count", 0)
        return {"model_call_count": count + 1}

    def wrap_tool_call(self, request, handler):
        result = handler(request)
        # 不能直接返回 state 更新，需在 node-style 钩子中
        return result

# 传给 create_deep_agent
agent = create_deep_agent(
    model=llm,
    state_schema=MyAgentState,
    middleware=[CountingMiddleware()],
)
```

## 三、注意事项

| 规则 | 说明 |
|------|------|
| `NotRequired` 必用 | 自定义字段必须声明为 `NotRequired`，否则初始 state 缺字段报错 |
| 不要在 `__init__` 后改属性 | 用 graph state 而非实例属性，避免并发竞态 |
| node-style 钩子返回 dict | `before_model`/`after_model` 返回 dict 合并到 state |
| wrap-style 钩子用 Command | `wrap_model_call`/`wrap_tool_call` 返回 `ExtendedModelResponse` + `Command` |

## 四、相关页面

- [[deepagents/customization/自定义概述]] — 其他定制方式
- [[langchain/middleware/自定义中间件开发]] — 中间件中的 state 管理
- [[langchain/agents/Agent循环实现]] — AgentState 底层原理
