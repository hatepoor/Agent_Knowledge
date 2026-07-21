---
tags: [langchain, middleware, langgraph, integration]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware
langchain_version: ">=1.0"
---

# 中间件与 LangGraph 的关系

## 一、核心结论

> **中间件不是独立运行时——它是 LangGraph 的"包装层"**。`create_agent()` 内部把 LangGraph 编译时插入钩子节点，中间件就是这些钩子节点上的函数。

```
中间件 = LangGraph 节点上的回调函数
create_agent(middleware=[...]) = 在 LangGraph 图中插入钩子节点 + 绑定回调
```

**理解这一点对进阶使用至关重要**：你既可以：
1. **直接用 `create_agent`**：让 LangChain 自动包装
2. **在自定义 LangGraph 中使用中间件**：把中间件逻辑直接嵌入图节点

## 二、create_agent 内部是怎么"插入"中间件的

`create_agent` 编译出的 LangGraph 图大致结构：

```python
# create_agent 内部伪代码
def create_agent(model, tools, middleware=[], ...):
    # 1. 构建 StateGraph
    graph = StateGraph(AgentState)

    # 2. 为每个 before_* 钩子创建包装节点
    for mw in middleware:
        if hasattr(mw, 'before_agent'):
            graph.add_node(f"before_agent_{mw.name}", mw.before_agent)
            graph.add_edge(START, f"before_agent_{mw.name}")

    # 3. 核心节点
    graph.add_node("agent", call_model)
    graph.add_node("tools", ToolNode(tools))

    # 4. 为每个 wrap_* 钩子嵌套包装
    wrapped_model = chain_wrap_middlewares(model, middleware)

    # 5. 编译
    return graph.compile()
```

**对你透明的部分**：
- 中间件节点的位置和边的连接
- wrap-style 中间件的嵌套顺序
- 状态合并的 reducer 规则

**你需要关心的**：
- 中间件列表的顺序（决定执行顺序）
- 状态字段的声明（`state_schema`）
- 异步实现（如果需要）

## 三、在 LangGraph 中嵌入 Agent

**场景**：你有一个更大的 LangGraph 工作流，希望把整个 Agent 作为子图嵌入。

```python
from langgraph.graph import START, StateGraph
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware

# 1. 创建带中间件的 Agent
email_agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[read_email, send_email],
    middleware=[
        HumanInTheLoopMiddleware(interrupt_on={"send_email": True}),
    ],
)

# 2. 把 Agent 作为节点嵌入更大的图
class WorkflowState(TypedDict):
    user_input: str
    classification: str
    email_response: str

workflow = (
    StateGraph(WorkflowState)
    .add_node("classify", classify_intent)        # 自定义节点：分类
    .add_node("email_agent", email_agent)          # 嵌入 Agent（含中间件）
    .add_node("respond", send_response)            # 自定义节点：回复
    .add_edge(START, "classify")
    .add_conditional_edges(
        "classify",
        lambda s: "email" if s["classification"] == "email" else "respond",
    )
    .compile()
)

# 整个 workflow.invoke 会触发 email_agent 的所有中间件
result = workflow.invoke({"user_input": "查看我的邮件"})
```

**关键点**：
- Agent 的中间件**仍然生效**
- HITL 中断会**传递到父图**——`workflow.invoke` 也会返回 `__interrupt__`
- Checkpointer 在父图和子图之间共享（thread_id 一致）

## 四、在 LangGraph 中复用中间件钩子（不通过 create_agent）

如果你自己写 LangGraph 图，想用类似中间件的能力：

### 4.1 用装饰器版的中间件作为 LangGraph 节点

```python
from langgraph.graph import StateGraph, START, END
from langchain.agents.middleware import before_model, AgentState
from langgraph.runtime import Runtime

@before_model
def log_step(state: AgentState, runtime: Runtime):
    print(f"调模型前: {len(state['messages'])} 条消息")
    return None

@before_model
def truncate_history(state, runtime):
    if len(state["messages"]) > 50:
        return {"messages": state["messages"][-50:]}
    return None

# 在自定义 LangGraph 中使用
class MyState(AgentState):
    pass

def call_model_node(state):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

graph = (
    StateGraph(MyState)
    .add_node("log_step", log_step)
    .add_node("truncate", truncate_history)
    .add_node("model", call_model_node)
    .add_edge(START, "log_step")
    .add_edge("log_step", "truncate")
    .add_edge("truncate", "model")
    .compile()
)
```

**注意**：装饰器版的中间件既可以在 `create_agent` 中用，也可以在自定义 LangGraph 中用——它们本质上是普通函数。

### 4.2 类方式中间件：在 LangGraph 中手动调用

```python
from langchain.agents.middleware import AgentMiddleware

class MyMiddleware(AgentMiddleware):
    def before_model(self, state, runtime):
        return {"messages": state["messages"][-30:]}

mw = MyMiddleware()

def my_node(state, runtime):
    # 手动调用中间件钩子
    update = mw.before_model(state, runtime)
    if update:
        state = {**state, **update}
    # 核心逻辑
    response = llm.invoke(state["messages"])
    return {"messages": [response]}
```

**优点**：完全控制中间件的执行位置和顺序  
**缺点**：失去 `create_agent` 的自动化（不推荐，复杂）

### 4.3 wrap-style 中间件在 LangGraph 中的使用

```python
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable

@wrap_model_call
def add_caching(request, handler):
    # 简单内存缓存
    cache_key = str(request.messages[-1].content)
    if cache_key in cache:
        return cache[cache_key]
    response = handler(request)
    cache[cache_key] = response
    return response

# 在 LangGraph 节点中应用
def cached_model_node(state, runtime):
    request = ModelRequest(
        messages=state["messages"],
        tools=state.get("tools", []),
        system_message=...,
    )
    response = add_caching(request, handler=lambda r: ModelResponse(...))
    return {"messages": [response.message]}
```

## 五、跨平台互操作

### 5.1 自定义 LangGraph 节点使用 LangChain 中间件

**场景**：你已经有一个 LangGraph 应用，想加入 LangChain 中间件能力

```python
# 导入中间件
from langchain.agents.middleware import PIIMiddleware, SummarizationMiddleware

# 1. 类方式中间件：作为 LangGraph 节点
pii = PIIMiddleware(pii_type="email", strategy="redact")
summary = SummarizationMiddleware(model=summary_llm, trigger=("tokens", 4000))

graph = (
    StateGraph(...)
    .add_node("redact_pii", lambda state, runtime: pii.before_model(state, runtime) or {})
    .add_node("model", call_model)
    .add_node("summarize", lambda state, runtime: summary.before_model(state, runtime) or {})
    .add_edge(START, "redact_pii")
    .add_edge("redact_pii", "model")
    .add_edge("model", "summarize")
    .add_edge("summarize", END)
    .compile()
)
```

### 5.2 第三方 LangGraph 库

许多第三方库（如 `deepagents`）就是基于 LangGraph + 中间件构建的。它们的实现方式：

```python
# 伪代码：deepagents 内部
class DeepAgent:
    def __init__(self, ...):
        self.graph = (
            StateGraph(...)
            .add_node("model", call_model)
            .add_node("files", FilesystemMiddleware())
            .add_node("subagent", SubAgentMiddleware())
            .add_node("todo", TodoListMiddleware())
            ...
            .compile()
        )
```

**启示**：中间件本质就是图节点。任何 LangGraph 应用都可以用中间件模式。

## 六、与 LangGraph 节点的对应关系

| 中间件钩子 | LangGraph 等价物 |
|-----------|-----------------|
| `before_agent` | `add_edge(START, "middleware_node")` |
| `before_model` | 模型节点前的普通节点 |
| `wrap_model_call` | 模型节点内的"嵌套包装"（不是图节点） |
| `after_model` | 模型节点后的普通节点 |
| `wrap_tool_call` | 工具节点内的"嵌套包装"（不是图节点） |
| `after_agent` | `add_edge("middleware_node", END)` |

**核心区别**：
- **Node-style 钩子** = LangGraph 节点（顺序执行）
- **Wrap-style 钩子** = 节点内的包装函数（嵌套执行）

## 七、什么时候用 create_agent vs 自定义 LangGraph

| 场景 | 推荐 |
|------|------|
| 标准 ReAct Agent | `create_agent` + 中间件 |
| 多 Agent 协作 | `create_agent` 多次 + LangGraph 编排 |
| 复杂状态机 | 自定义 LangGraph（用中间件做横切关注点） |
| 条件分支、循环 | 自定义 LangGraph |
| 标准多轮对话 | `create_agent` + checkpointer |
| 流式输出 / 实时响应 | 都支持 |

## 八、调试混合架构

当你在 LangGraph 中嵌入 Agent 时调试：

```python
# 启用 LangGraph 调试
graph.get_graph().print_ascii()

# 输出：
#         ┌──────────────┐
#         │    classify   │
#         └──────┬───────┘
#                │
#        ┌───────┴────────┐
#        ▼                ▼
# ┌─────────────┐  ┌──────────────┐
# │  email_agent│  │   respond    │
# └─────────────┘  └──────────────┘
#  (内部含 HITL 中间件)
```

```python
# 查看 stream 中的中间件事件
async for event in graph.astream_events(input, config, version="v3"):
    if "email_agent" in str(event.get("name", "")):
        print(f"[子 Agent 事件] {event['event']}")
```

## 九、相关页面

- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/自定义中间件开发]] — 编写自定义中间件
- [[langchain/middleware/最佳实践]] — 中间件组合策略
- [[langgraph/overview/LangGraph概述]] — LangGraph 基础
- [[langgraph/modules/LangGraph模块详解]] — LangGraph 模块详解
