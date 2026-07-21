---
tags: [langchain, agent, langgraph, subgraph]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# Agent 与 LangGraph 的关系

## 概述

`create_agent` 返回的对象并非普通函数，而是**编译好的 `CompiledStateGraph`**。这意味着 Agent 本身就是一个完整的 LangGraph 图，可以直接作为子图嵌入更大的工作流。

## 核心概念

- Agent 的底层执行引擎是 LangGraph 的状态图
- `create_agent` 封装了模型调用循环、工具调度、消息传递等逻辑为一个图
- 返回的 `CompiledStateGraph` 支持所有标准图操作：`invoke`、`stream`、`batch`、`ainvoke` 等

## 子图嵌入示例

将整个 Agent 作为更大工作流中的一个节点：

```python
from langgraph.graph import StateGraph, START

# 把整个 agent 当作子图
graph = (
    StateGraph(dict)
    .add_node("agent", agent)
    .add_edge(START, "agent")
    .compile()
)
```

## 多 Agent 协作场景

由于每个 Agent 都是一个图节点，可以轻松实现多 Agent 协作：

```python
from langgraph.graph import StateGraph, START, END

graph = (
    StateGraph(dict)
    .add_node("researcher", research_agent)
    .add_node("writer", writer_agent)
    .add_edge(START, "researcher")
    .add_edge("researcher", "writer")
    .add_edge("writer", END)
    .compile()
)
```

## 与普通图节点的区别

| 特性 | 普通图节点 | Agent 子图 |
|------|-----------|-----------|
| 内部循环 | 无 | 有（模型-工具循环） |
| 工具调用 | 手动实现 | 自动调度 |
| 状态管理 | 自定义 | 内置 AgentState |
| 终止条件 | 自定义 | 模型自主决定 |

## 相关页面

- [[langchain/modules/langchain_agents/概述|← 返回 agents 概述]]
- [[langchain/modules/langchain_agents/create_agent参数|create_agent 参数详解]]
- [[langchain/modules/langchain_checkpoints/概述|checkpoints 模块]] — 状态持久化
