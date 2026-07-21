---
tags: [langgraph, react, agent, 教程, 工具调用, 条件边]
type: tutorial
created: 2026-07-11
updated: 2026-07-11
---

# ReAct Agent 实现

## 什么是 ReAct

**ReAct**（Reason + Act）是一种让大语言模型交替进行**推理**和**行动**的范式。其核心思想是：

1. **Reason（推理）**：模型分析当前状态，决定下一步该做什么
2. **Act（行动）**：执行工具调用，获取外部信息
3. **Observe（观察）**：将工具返回的结果加入上下文
4. **循环**：重复以上步骤，直到模型认为可以给出最终答案

```
┌──────────────────────────────────────────┐
│                                          │
│   用户输入                                │
│     │                                    │
│     ▼                                    │
│  ┌─────────┐    有工具调用    ┌────────┐  │
│  │  Agent   │ ──────────────→│  Tools  │  │
│  │ (推理)   │                │ (行动)  │  │
│  └─────────┘ ←────────────── └────────┘  │
│     │            工具结果                 │
│     │ 无工具调用                          │
│     ▼                                    │
│   最终回答                                │
│                                          │
└──────────────────────────────────────────┘
```

这个循环结构正是 LangGraph 擅长表达的——**有向循环图**。

## 前置准备

```bash
pip install langgraph langchain-openai langchain-community
```

```python
import os
os.environ["OPENAI_API_KEY"] = "your-api-key-here"
```

## 第一步：定义状态

ReAct Agent 的状态核心是**消息列表**。所有节点都围绕消息进行操作：

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    # 消息列表，使用 add_messages reducer 自动追加新消息
    messages: Annotated[list, add_messages][[
```

> **为什么用 `add_messages`？** 默认的状态更新是覆盖模式，但消息列表需要的是追加模式。`add_messages` 这个 Reducer 会自动将新消息追加到列表中，并处理去重。详见 [[langgraph/modules/langgraph_state/Reducer|Reducer 归约函数]]。

## 第二步：定义工具

创建一个简单的搜索工具：

```python
from langchain_core.tools import tool

@tool
def search_web(query: str) -> str:
    """搜索网络获取实时信息。输入应该是搜索关键词。"""
    # 这里用模拟数据代替真实搜索 API
    fake_results = {
        "langgraph": "LangGraph 是 LangChain 的图计算框架，用于构建有状态的 Agent 应用。最新版本支持多 Agent 协作和流式处理。",
        "python": "Python 是一种广泛使用的高级编程语言，以简洁易读著称。当前最新稳定版本为 3.12。",
        "天气": "今日北京天气晴，气温 25°C，适合户外活动。",
    }
    # 简单关键词匹配
    for key, value in fake_results.items():
        if key in query.lower():
            return value
    return f"未找到关于 '{query}' 的相关信息。"

@tool
def calculate(expression: str) -> str:
    """计算数学表达式。输入应该是合法的 Python 数学表达式。"""
    try:
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"计算错误: {e}"

tools = [search_web, calculate]
```

## 第三步：创建 Agent 节点（推理节点）

Agent 节点负责调用模型进行推理，决定是否使用工具：

```python
from langchain_openai import ChatOpenAI

# 创建模型并绑定工具
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
llm_with_tools = llm.bind_tools(tools)

def agent_node(state: AgentState) -> dict:
    """Agent 推理节点：调用模型决定是否使用工具"""
    messages = state["messages"]
    response = llm_with_tools.invoke(messages)
    # 返回的消息会被 add_messages reducer 追加到状态中
    return {"messages": [response]}
```

## 第四步：创建工具节点（行动节点）

工具节点负责执行模型请求的工具调用：

```python
from langgraph.prebuilt import ToolNode

# 方式一：使用 LangGraph 内置的 ToolNode（推荐）
tool_node = ToolNode(tools)

# 方式二：手动实现工具节点（帮助理解原理）
def manual_tool_node(state: AgentState) -> dict:
    """手动实现工具执行节点"""
    messages = state["messages"]
    # 获取最后一条消息中的工具调用请求
    last_message = messages[-1]
    tool_calls = last_message.tool_calls
    
    # 执行每个工具调用
    results = [][[
    for tool_call in tool_calls:
        # 找到对应的工具
        tool_map = {t.name: t for t in tools}
        selected_tool = tool_map[tool_call["name"]]
        # 执行工具
        result = selected_tool.invoke(tool_call["args"])
        # 构造 ToolMessage 返回
        from langchain_core.messages import ToolMessage
        results.append(
            ToolMessage(
                content=result,
                name=tool_call["name"],
                tool_call_id=tool_call["id"],
            )
        )
    return {"messages": results}
```

> **ToolNode 的工作原理**：它会自动检查最后一条消息中的 `tool_calls` 字段，找到对应的工具并执行，然后将结果包装为 `ToolMessage` 返回。

## 第五步：添加条件边（路由决策）

条件边是 ReAct 循环的关键——它决定是继续执行工具还是返回最终答案：

```python
def should_continue(state: AgentState) -> str:
    """判断是否应该继续调用工具
    
    检查最后一条消息是否包含工具调用请求：
    - 有工具调用 → 路由到 "tools" 节点
    - 无工具调用 → 路由到 END（结束）
    """
    last_message = state["messages"][-1]
    # 如果消息中包含 tool_calls，说明模型想使用工具
    if last_message.tool_calls:
        return "continue"  # 继续执行工具
    else:
        return "end"  # 结束循环，返回最终答案
```

## 第六步：构建完整的图

将所有节点和边组装起来：

```python
from langgraph.graph import StateGraph, START, END

# 创建状态图
workflow = StateGraph(AgentState)

# 添加节点
workflow.add_node("agent", agent_node)
workflow.add_node("tools", tool_node)

# 添加边
workflow.add_edge(START, "agent")           # 入口 → Agent 推理
workflow.add_conditional_edges(             # Agent → 条件判断
    "agent",
    should_continue,
    {
        "continue": "tools",   # 需要工具 → 执行工具
        "end": END,            # 不需要 → 结束
    }
)
workflow.add_edge("tools", "agent")         # 工具执行完 → 回到 Agent 继续推理

# 编译图
app = workflow.compile()
```

图的完整结构：

```
         ┌──────────────────────────┐
         │                          │
         ▼                          │
START → agent ──→ [条件判断]        │
                    │               │
              ┌─────┴─────┐        │
              │           │        │
         continue       end        │
              │           │        │
              ▼           ▼        │
           tools ────→ END    (回到 agent)
```

## 完整可运行代码

```python
"""
ReAct Agent 完整实现
从零构建一个具备搜索和计算能力的 AI Agent
"""
import os
from typing import TypedDict, Annotated

from langchain_core.tools import tool
from langchain_core.messages import HumanMessage
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode

# ========== 1. 定义状态 ==========
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]

# ========== 2. 定义工具 ==========
@tool
def search_web(query: str) -> str:
    """搜索网络获取实时信息。输入应该是搜索关键词。"""
    fake_results = {
        "langgraph": "LangGraph 是 LangChain 的图计算框架，用于构建有状态的 Agent 应用。",
        "python": "Python 是一种广泛使用的高级编程语言，当前最新稳定版本为 3.12。",
        "天气": "今日北京天气晴，气温 25°C，适合户外活动。",
    }
    for key, value in fake_results.items():
        if key in query.lower():
            return value
    return f"未找到关于 '{query}' 的相关信息。"

@tool
def calculate(expression: str) -> str:
    """计算数学表达式。输入应该是合法的 Python 数学表达式。"""
    try:
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"计算错误: {e}"

tools = [search_web, calculate]

# ========== 3. 创建模型 ==========
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
llm_with_tools = llm.bind_tools(tools)

# ========== 4. 定义节点 ==========
def agent_node(state: AgentState) -> dict:
    """Agent 推理节点"""
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

tool_node = ToolNode(tools)

# ========== 5. 条件路由 ==========
def should_continue(state: AgentState) -> str:
    """判断是否继续调用工具"""
    if state["messages"][-1].tool_calls:
        return "continue"
    return "end"

# ========== 6. 构建图 ==========
workflow = StateGraph(AgentState)
workflow.add_node("agent", agent_node)
workflow.add_node("tools", tool_node)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges("agent", should_continue, {"continue": "tools", "end": END})
workflow.add_edge("tools", "agent")

app = workflow.compile()

# ========== 7. 运行测试 ==========
if __name__ == "__main__":
    # 测试搜索功能
    result = app.invoke({
        "messages": [HumanMessage(content="帮我搜索一下 LangGraph 是什么？")]
    })
    print("=== 搜索测试 ===")
    print(result["messages"][-1].content)
    
    # 测试计算功能
    result = app.invoke({
        "messages": [HumanMessage(content="计算 123 * 456 + 789 的结果")]
    })
    print("\n=== 计算测试 ===")
    print(result["messages"][-1].content)
```

## 执行流程详解

以用户提问"帮我搜索一下 LangGraph 是什么？"为例，完整执行流程如下：

```
步骤 1: START → agent
┌─────────────────────────────────────────────┐
│ 状态: messages = [HumanMessage("搜索...")]    │
│ 动作: 调用 llm_with_tools.invoke(messages)   │
│ 结果: AIMessage(tool_calls=[                 │
│         {name: "search_web",                 │
│          args: {query: "LangGraph"}}         │
│       ])                                     │
└─────────────────────────────────────────────┘
                    │
                    ▼
步骤 2: agent → should_continue → "continue"
┌─────────────────────────────────────────────┐
│ 判断: 最后一条消息有 tool_calls？              │
│ 结果: 是 → 路由到 tools 节点                  │
└─────────────────────────────────────────────┘
                    │
                    ▼
步骤 3: tools → agent
┌─────────────────────────────────────────────┐
│ 状态: 执行 search_web("LangGraph")           │
│ 结果: ToolMessage("LangGraph 是 LangChain...")│
│ 追加到 messages 后，回到 agent 节点           │
└─────────────────────────────────────────────┘
                    │
                    ▼
步骤 4: agent → should_continue → "end"
┌─────────────────────────────────────────────┐
│ 动作: 模型看到工具结果，生成最终回答            │
│ 结果: AIMessage("LangGraph 是...")           │
│ 判断: 没有 tool_calls → 路由到 END            │
└─────────────────────────────────────────────┘
                    │
                    ▼
步骤 5: END → 返回最终状态
```

## 流式输出

使用 `stream` 方法可以实时观察 Agent 的执行过程：

```python
from langchain_core.messages import HumanMessage

# 使用 stream 观察执行过程
for event in app.stream(
    {"messages": [HumanMessage(content="今天北京天气怎么样？")]},
    stream_mode="updates"  # 只看每个节点的增量输出
):
    for node_name, output in event.items():
        print(f"\n--- 节点: {node_name} ---")
        for msg in output.get("messages", []):
            print(f"  {msg.__class__.__name__}: {msg.content[:100]}")
```

输出示例：

```
--- 节点: agent ---
  AIMessage: (包含 tool_calls)

--- 节点: tools ---
  ToolMessage: 今日北京天气晴，气温 25°C，适合户外活动。

--- 节点: agent ---
  AIMessage: 今天北京天气晴朗，气温 25°C，非常适合户外活动。
```

## 添加系统提示词

系统提示词可以控制 Agent 的行为风格：

```python
from langchain_core.messages import SystemMessage

def agent_node(state: AgentState) -> dict:
    """带系统提示词的 Agent 推理节点"""
    system_prompt = SystemMessage(content=(
        "你是一个乐于助人的 AI 助手。\n"
        "当用户的问题需要实时信息或计算时，请使用工具。\n"
        "回答要简洁、准确。"
    ))
    messages = [system_prompt] + state["messages"]
    return {"messages": [llm_with_tools.invoke(messages)]}
```

## 与 LangChain create_agent 的对比

LangChain 的 `create_agent` 函数内部就是使用 LangGraph 实现的：

```python
# 方式一：使用 LangChain 高层 API（内部使用 LangGraph）
from langchain.agents import create_agent

agent = create_agent(model="gpt-4o-mini", tools=tools)
result = agent.invoke({
    "messages": [HumanMessage(content="搜索 LangGraph")]
})

# 方式二：手动用 LangGraph 构建（本教程的方式）
# ... 就是上面完整的代码 ...
```

| 对比维度 | `create_agent` | 手动 LangGraph |
|---------|----------------|----------------|
| 代码量 | 极少（一行创建） | 较多（需要手动定义每个组件） |
| 灵活性 | 有限 | 完全自定义 |
| 可控性 | 黑盒 | 白盒，每个节点可调试 |
| 适用场景 | 标准 ReAct Agent | 需要自定义流程的复杂场景 |
| 底层实现 | LangGraph StateGraph | 就是 LangGraph 本身 |
| 条件路由 | 内置 should_continue | 自定义任意路由逻辑 |
| 状态管理 | 固定 AgentState | 自定义任意状态结构 |

> **建议**：简单的 ReAct Agent 直接用 `create_agent`；需要自定义流程（如添加审批节点、多 Agent 协作、复杂条件路由）时，手动用 LangGraph 构建。

## 常见问题

### 为什么需要循环？

因为模型可能需要多次工具调用才能收集到足够的信息。例如：
- 用户问"LangGraph 和 LangChain 有什么区别？"
- 模型可能先搜索 "LangGraph"，再搜索 "LangChain"，最后综合回答

### 如何限制最大循环次数？

```python
# 编译时设置递归限制
app = workflow.compile()

# 或者在 invoke 时设置
result = app.invoke(
    {"messages": [HumanMessage(content="...")][[},
    config={"recursion_limit": 10}  # 最多 10 次循环
)
```

### 如何处理工具执行错误？

```python
@tool
def safe_tool(query: str) -> str:
    """安全的工具，自带错误处理"""
    try:
        # 实际逻辑
        return do_something(query)
    except Exception as e:
        return f"工具执行出错: {e}，请尝试其他方式回答用户的问题。"
```

## 相关页面

- [[langgraph/concepts/图计算范式|图计算范式]] — 为什么 Agent 需要图结构
- [[langgraph/concepts/LangGraph_vs_LangChain|LangGraph vs LangChain]] — 两者对比
- [[langgraph/tutorial/多轮对话Agent|多轮对话 Agent]] — 添加记忆功能
- [[langgraph/tutorial/RAG_Agent|RAG Agent]] — 结合检索增强生成
- [[langgraph/modules/langgraph_state/概述|状态管理概述]] — 状态系统详解
- [[langchain/modules/langchain_agents/Agent与LangGraph关系|Agent 与 LangGraph 关系]]
