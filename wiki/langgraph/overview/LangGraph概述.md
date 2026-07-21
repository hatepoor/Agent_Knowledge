---
tags: [langgraph, overview, agent, orchestration]
type: overview
created: 2026-07-11
updated: 2026-07-11
---

# LangGraph 概述

LangGraph 是一个**低级编排框架**，用于构建有状态的、支持循环的 AI Agent 和多步工作流。它将工作流建模为**图（Graph）**——由节点、边和共享状态组成——让你可以精确控制 Agent 的执行逻辑。

## 核心思想

**将工作流建模为图。** LangGraph 的核心抽象是：

- **节点（Node）**：执行具体操作的函数（调用模型、运行工具、处理数据）
- **边（Edge）**：定义节点之间的执行顺序和路由逻辑
- **状态（State）**：在节点之间流动和更新的共享数据

```
         ┌─────────────┐
         │   START     │
         └──────┬──────┘
                │
         ┌──────▼──────┐
         │  接收输入    │
         └──────┬──────┘
                │
         ┌──────▼──────┐
    ┌───►│  推理节点    │◄──── 条件边
    │    └──────┬──────┘       （循环）
    │           │
    │    ┌──────▼──────┐
    └────│  工具节点    │
         └──────┬──────┘
                │
         ┌──────▼──────┐
         │    END      │
         └─────────────┘
```

## 为什么需要图

传统 LLM 应用是线性的（Chain），但真实世界的 Agent 需要：

| 需求 | 线性 Chain | LangGraph 图 |
|------|-----------|-------------|
| 循环执行（Agent 反复调用工具） | 不支持 | 原生支持 |
| 条件路由（根据状态选择分支） | 需要硬编码 | 条件边 |
| 持久化状态（跨步骤保持上下文） | 手动管理 | 内置状态管理 |
| 人机协作（人工审批节点） | 难以实现 | 中断与恢复 |
| 多 Agent 协作 | 几乎不可能 | 子图与多 Agent 模式 |

## 与 LangChain 的关系

LangGraph 和 LangChain 是互补的，不是替代的：

```
┌─────────────────────────────────────────────┐
│              你的 AI 应用                     │
├─────────────────────────────────────────────┤
│  LangGraph：编排层                            │
│  - 定义工作流结构（图）                        │
│  - 管理状态流转                               │
│  - 控制执行逻辑                               │
├─────────────────────────────────────────────┤
│  LangChain：组件层                            │
│  - 提供 LLM 模型接口                         │
│  - 提供工具（Tools）                          │
│  - 提供 Prompt 模板                           │
│  - 提供输出解析器                              │
└─────────────────────────────────────────────┘
```

**简单类比**：
- LangChain = 乐高积木（模型、工具、提示词等组件）
- LangGraph = 乐高说明书（如何组装这些积木成完整应用）

## 核心特性

### 1. 持久化（Persistence）

通过 Checkpointer 自动保存每一步的状态，支持：
- 对话恢复（断点续传）
- 时间旅行（回溯到任意历史状态）
- 跨会话记忆

```python
from langgraph.checkpoint.memory import MemorySaver

# 使用内存 Checkpointer
checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# 通过 thread_id 标识对话
config = {"configurable": {"thread_id": "user-123"}}
result = graph.invoke({"messages": [("user", "你好")]}, config)
```

### 2. 流式传输（Streaming）

支持多种流式模式：
- **values**：每步输出完整状态
- **updates**：每步输出状态增量
- **messages**：流式输出 LLM 生成的 token

```python
# 流式输出每一步的状态更新
for step in graph.stream(inputs, stream_mode="updates"):
    print(step)
```

### 3. 人机协作（Human-in-the-Loop）

在关键节点暂停执行，等待人工审批或输入：

```python
from langgraph.types import interrupt

def approve_action(state):
    # 暂停并等待人工确认
    human_feedback = interrupt({"question": "是否执行此操作？"})
    if human_feedback == "approve":
        return {"status": "approved"}
    return {"status": "rejected"}
```

### 4. 多 Agent 系统

支持多种多 Agent 协作模式：
- **监督者模式**：一个 Agent 协调多个专业 Agent
- **协作模式**：Agent 之间平等协作
- **层级模式**：多层级的 Agent 组织

### 5. 子图（Subgraphs）

将复杂工作流拆分为可复用的子图：

```python
# 创建子图
subgraph = StateGraph(SubState)
subgraph.add_node("process", process_node)
subgraph.add_edge(START, "process")
subgraph.add_edge("process", END)

# 在主图中使用子图
main_graph = StateGraph(MainState)
main_graph.add_node("subtask", subgraph.compile())
```

## 包结构

LangGraph 由多个包组成：

| 包名 | 作用 | 安装 |
|------|------|------|
| `langgraph` | 核心图引擎 | `pip install langgraph` |
| `langgraph-checkpoint` | Checkpoint 抽象接口 | 随 langgraph 安装 |
| `langgraph-checkpoint-sqlite` | SQLite 持久化后端 | `pip install langgraph-checkpoint-sqlite` |
| `langgraph-checkpoint-postgres` | PostgreSQL 持久化后端 | `pip install langgraph-checkpoint-postgres` |
| `langgraph-sdk` | 与 LangGraph Platform 交互的客户端 | `pip install langgraph-sdk` |

## 最简示例

一个只有两个节点和一条边的最简图：

```python
"""
最简 LangGraph 示例：两个节点 + 一条边
"""
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

# 1. 定义状态结构
class State(TypedDict):
    text: str

# 2. 定义节点函数
def greet(state: State) -> dict:
    """问候节点：添加问候语"""
    return {"text": f"你好！{state['text']}"}

def farewell(state: State) -> dict:
    """告别节点：添加告别语"""
    return {"text": f"{state['text']} 再见！"}

# 3. 构建图
builder = StateGraph(State)
builder.add_node("greet", greet)
builder.add_node("farewell", farewell)

# 4. 添加边：START → greet → farewell → END
builder.add_edge(START, "greet")
builder.add_edge("greet", "farewell")
builder.add_edge("farewell", END)

# 5. 编译并执行
graph = builder.compile()
result = graph.invoke({"text": "欢迎使用 LangGraph"})
print(result["text"][[)  # 输出：你好！欢迎使用 LangGraph 再见！
```

## 何时使用 LangGraph vs 纯 LangChain

| 场景 | 推荐方案 |
|------|---------|
| 简单的单次 LLM 调用 | LangChain |
| 简单的 RAG 管道 | LangChain |
| 需要循环的 Agent（反复调用工具） | **LangGraph** |
| 复杂的多步工作流 | **LangGraph** |
| 需要人机协作 | **LangGraph** |
| 多 Agent 系统 | **LangGraph** |
| 需要状态持久化 | **LangGraph** |
| 需要精细控制执行流程 | **LangGraph** |

**经验法则**：
- 如果你的工作流是**线性的、无状态的**，用 LangChain
- 如果你的工作流需要**循环、条件分支、状态管理**，用 LangGraph

## 相关页面

- [[langgraph/quickstart/快速入门|快速入门]] — 从零开始构建你的第一个 LangGraph 应用
- [[langgraph/modules/LangGraph模块详解|模块详解]] — 所有模块的导航页
- [[langgraph/modules/langgraph_core/概述|核心模块]] — StateGraph、节点、边的详细 API
- [[langchain/overview/LangChain概述|LangChain 概述]] — LangChain 在 Agent 生态中的定位
