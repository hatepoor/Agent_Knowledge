---
tags: [langgraph, langchain, 对比, 概念, 架构]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# LangGraph vs LangChain

## 概述

LangGraph 和 LangChain 是同一生态中的两个互补框架。简单来说：

- **LangChain** 提供**组件**：模型、工具、提示词、解析器、向量数据库等
- **LangGraph** 提供**运行时**：编排、状态管理、持久化、流式处理

它们不是竞争关系，而是**协同工作**。实际上，LangChain 的 `create_agent` 函数内部就是用 LangGraph 实现的。

```
┌─────────────────────────────────────────────────────┐
│                   你的应用                            │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │           LangGraph（编排层）                 │    │
│  │                                             │    │
│  │  状态管理 · 图编排 · 条件路由 · 持久化 · 流式  │    │
│  │                                             │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐       │    │
│  │  │ 节点 A  │ │ 节点 B  │ │ 节点 C  │       │    │
│  │  └────┬────┘ └────┬────┘ └────┬────┘       │    │
│  └───────┼───────────┼───────────┼────────────┘    │
│          │           │           │                  │
│  ┌───────┼───────────┼───────────┼────────────┐    │
│  │       ▼           ▼           ▼            │    │
│  │           LangChain（组件层）               │    │
│  │                                           │    │
│  │  模型 · 工具 · 提示词 · 解析器 · 向量库     │    │
│  │                                           │    │
│  └───────────────────────────────────────────┘    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## LangChain：组件工具箱

LangChain 的核心价值是提供了一套**标准化的组件接口**，让你可以轻松替换和组合不同的 LLM 组件。

### 核心组件

```python
# ===== 模型 =====
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

llm = ChatOpenAI(model="gpt-4o-mini")
# 可以轻松切换模型提供商
# llm = ChatAnthropic(model="claude-3-5-sonnet")

# ===== 工具 =====
from langchain_core.tools import tool

@tool
def search(query: str) -> str:
    """搜索网络"""
    return "搜索结果"

# ===== 提示词 =====
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}助手"),
    ("user", "{input}"),
])

# ===== 输出解析器 =====
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser

parser = StrOutputParser()

# ===== 向量数据库 =====
from langchain_chroma import Chroma
# ⚠️ langchain_community 已弃用
from langchain_community.vectorstores import FAISS

# ===== 文档处理 =====
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
```

### LCEL 链式组合

LangChain 使用 **LCEL（LangChain Expression Language）** 将组件串联成链：

```python
# LCEL：用管道符 | 组合组件
chain = prompt | llm | parser

# 数据从左到右流动
result = chain.invoke({"role": "编程", "input": "什么是 Python？"})
```

```
[Prompt] → [LLM] → [Parser] → 字符串输出
```

### LangChain 擅长的事

| 能力 | 说明 |
|------|------|
| 模型抽象 | 统一接口调用不同 LLM 提供商 |
| 工具定义 | 标准化的 `@tool` 装饰器 |
| 提示词管理 | 模板、变量、消息构造 |
| 文档处理 | 加载、分割、嵌入、存储 |
| 检索 | 向量检索、混合检索、重排序 |
| 输出解析 | 字符串、JSON、结构化输出 |
| 回调追踪 | LangSmith 集成、执行追踪 |

### LangChain 不擅长的事

| 局限 | 说明 |
|------|------|
| 循环流程 | LCEL 是 DAG，无法表达循环 |
| 状态管理 | 没有内置的全局状态机制 |
| 持久化 | 无法自动保存和恢复执行状态 |
| 条件路由 | 复杂的动态路由难以实现 |
| 多 Agent | 多 Agent 协作需要额外框架 |

## LangGraph：运行时引擎

LangGraph 的核心价值是提供了一个**图执行引擎**，让你能够编排复杂的、有状态的 AI 工作流。

### 核心组件

```python
# ===== 状态定义 =====
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    context: str

# ===== 图构建 =====
from langgraph.graph import StateGraph, START, END

workflow = StateGraph(AgentState)
workflow.add_node("agent", agent_node)
workflow.add_node("tools", tool_node)

# ===== 边和条件路由 =====
workflow.add_edge(START, "agent")
workflow.add_conditional_edges("agent", should_continue, {
    "continue": "tools",
    "end": END,
})
workflow.add_edge("tools", "agent")

# ===== 持久化 =====
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
app = workflow.compile(checkpointer=checkpointer)

# ===== 执行 =====
result = app.invoke(
    {"messages": [HumanMessage(content="你好")]},
    config={"configurable": {"thread_id": "thread_1"}}
)
```

### LangGraph 擅长的事

| 能力 | 说明 |
|------|------|
| 循环编排 | 支持 Agent 的推理-行动循环 |
| 状态管理 | 全局共享状态 + Reducer 合并 |
| 持久化 | Checkpointer 自动保存/恢复状态 |
| 条件路由 | 动态决定执行路径 |
| 多 Agent | 图嵌套、子图、Agent 间通信 |
| 人机交互 | 中断、审批、人工介入 |
| 流式处理 | 三种流式模式（values/updates/messages） |
| 时间旅行 | 回溯到任意历史状态 |

### LangGraph 不擅长的事

| 局限 | 说明 |
|------|------|
| 模型调用 | 不提供模型接口，需要 LangChain 的模型 |
| 工具定义 | 不提供工具装饰器，需要 LangChain 的 `@tool` |
| 提示词 | 不提供提示词模板，需要 LangChain 的 Prompt |
| 向量检索 | 不提供向量数据库，需要 LangChain 的 VectorStore |
| 文档处理 | 不提供加载器/分割器，需要 LangChain 的组件 |

## 对比总表

| 维度 | LangChain | LangGraph |
|------|-----------|-----------|
| **定位** | 组件库 / 工具箱 | 编排引擎 / 运行时 |
| **核心抽象** | Chain（链） | Graph（图） |
| **数据流** | 单向（DAG） | 可循环（有向循环图） |
| **状态管理** | 无内置机制 | 全局状态 + Reducer |
| **持久化** | 无 | Checkpointer（内存/SQLite/PostgreSQL） |
| **条件路由** | RunnableBranch（有限） | add_conditional_edges（灵活） |
| **循环支持** | 不支持 | 核心特性 |
| **多 Agent** | 不直接支持 | 图嵌套、子图 |
| **人机交互** | 不直接支持 | 内置中断/恢复 |
| **流式输出** | 支持（模型级） | 支持（节点级 + token 级） |
| **学习曲线** | 低-中 | 中-高 |
| **适用场景** | 简单链、RAG、格式化 | Agent、复杂工作流、多轮对话 |
| **包名** | `langchain`, `langchain-core` | `langgraph` |

## 它们如何协同工作

### 示例：LangGraph 图 + LangChain 组件

```python
# LangChain 组件
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import SystemMessage
from langchain_chroma import Chroma

# LangGraph 编排
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver

# ---- 使用 LangChain 的组件 ----

# 模型（来自 langchain-openai）
llm = ChatOpenAI(model="gpt-4o-mini")

# 工具（来自 langchain-core）
@tool
def search(query: str) -> str:
    """搜索网络"""
    return "搜索结果"

@tool
def query_knowledge_base(question: str) -> str:
    """查询内部知识库"""
    # 使用 LangChain 的向量数据库
    db = Chroma(...)
    docs = db.similarity_search(question)
    return "\n".join([d.page_content for d in docs])

tools = [search, query_knowledge_base]
llm_with_tools = llm.bind_tools(tools)

# ---- 使用 LangGraph 编排 ----

# 定义节点
def agent_node(state: MessagesState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

# 构建图
workflow = StateGraph(MessagesState)
workflow.add_node("agent", agent_node)
workflow.add_node("tools", ToolNode(tools))
workflow.add_edge(START, "agent")
workflow.add_conditional_edges("agent", should_continue, {
    "continue": "tools",
    "end": END,
})
workflow.add_edge("tools", "agent")

# 添加持久化
checkpointer = MemorySaver()
app = workflow.compile(checkpointer=checkpointer)

# 运行
result = app.invoke(
    {"messages": [HumanMessage(content="LangGraph 有什么特点？")]},
    config={"configurable": {"thread_id": "demo"}}
)
```

在这个例子中：
- **LangChain 提供**：ChatOpenAI（模型）、@tool（工具定义）、Chroma（向量数据库）
- **LangGraph 提供**：StateGraph（图编排）、MessagesState（状态管理）、MemorySaver（持久化）、条件路由

## create_agent 的秘密

LangChain 的 `create_agent` 实际上就是 LangGraph 的封装：

```python
# LangChain 高层 API
from langchain.agents import create_agent

agent = create_agent(model="gpt-4o-mini", tools=[search])

# 等价于以下 LangGraph 代码：
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4o-mini"),
    tools=[search][[,
)

# create_react_agent 内部就是：
# 1. 创建 StateGraph(MessagesState)
# 2. 添加 agent 节点（调用 LLM）
# 3. 添加 tools 节点（ToolNode）
# 4. 添加条件边（should_continue）
# 5. 编译并返回 CompiledStateGraph
```

> 详见 [[langchain/modules/langchain_agents/Agent与LangGraph关系|Agent 与 LangGraph 关系]]

## 选择指南

### 使用 LangChain（LCEL）就够了

```
✅ 简单的单次调用
✅ 固定流程的 RAG（检索 → 生成）
✅ 数据转换管道
✅ 简单的格式化/解析
✅ 不需要循环或条件分支
```

```python
# LCEL 就足够了
chain = prompt | llm | parser
result = chain.invoke({"input": "..."})
```

### 需要 LangGraph

```
✅ Agent（需要推理-行动循环）
✅ 多轮对话（需要记忆/持久化）
✅ 自我纠错（需要反馈循环）
✅ 多 Agent 协作
✅ 复杂条件路由
✅ 人机交互（审批、中断）
✅ 需要时间旅行/调试
```

```python
# 需要 LangGraph 的图编排
workflow = StateGraph(State)
workflow.add_node(...)
workflow.add_conditional_edges(...)
app = workflow.compile(checkpointer=...)
```

### 两者结合

```
✅ 大多数实际项目
✅ LangChain 组件 + LangGraph 编排
✅ create_agent（内部已经结合）
```

## 架构演进路径

```
阶段 1：简单应用
┌──────────────────────────┐
│  LangChain (LCEL)        │
│  prompt | llm | parser   │
│  适合：简单问答、格式化    │
└──────────────────────────┘

阶段 2：添加 RAG
┌──────────────────────────┐
│  LangChain (LCEL)        │
│  retriever | prompt |    │
│  llm | parser            │
│  适合：单次 RAG 问答      │
└──────────────────────────┘

阶段 3：需要 Agent
┌──────────────────────────────────────┐
│  LangGraph + LangChain 组件          │
│                                      │
│  StateGraph:                         │
│    agent → 条件判断 → tools → agent  │
│    (使用 LangChain 的模型和工具)      │
│  适合：ReAct Agent                   │
└──────────────────────────────────────┘

阶段 4：复杂系统
┌──────────────────────────────────────┐
│  LangGraph + LangChain + 多 Agent    │
│                                      │
│  StateGraph:                         │
│    协调者 → 研究员 Agent              │
│          → 写手 Agent                │
│          → 审核 Agent                │
│    (每个 Agent 都是子图)              │
│  适合：多 Agent 协作系统              │
└──────────────────────────────────────┘
```

## 常见误解

### 误解 1："LangGraph 会取代 LangChain"

**事实**：LangGraph 依赖 LangChain 的组件。它们在不同层次工作：
- LangChain 解决"用什么组件"
- LangGraph 解决"怎么编排组件"

### 误解 2："用了 create_agent 就不需要学 LangGraph"

**事实**：`create_agent` 只覆盖了标准 ReAct 模式。当你需要自定义流程时（添加审批节点、多 Agent 协作、复杂路由），必须理解 LangGraph。

### 误解 3："LangGraph 只能做 Agent"

**事实**：LangGraph 是通用的图编排引擎。任何需要状态管理、条件路由、持久化的工作流都可以用 LangGraph，不限于 Agent。

## 相关页面

- [[langgraph/concepts/图计算范式|图计算范式]] — 为什么需要图结构
- [[langgraph/tutorial/ReAct_Agent实现|ReAct Agent 实现]] — LangGraph 实际使用
- [[langchain/modules/langchain_agents/Agent与LangGraph关系|Agent 与 LangGraph 关系]] — create_agent 内部实现
- [[langchain/modules/langchain_agents/概述|LangChain Agents 概述]] — Agent 模块
- [[langgraph/modules/langgraph_state/概述|状态管理概述]] — LangGraph 状态系统
