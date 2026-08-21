---
tags: [langgraph, modules, navigation, index]
type: overview
created: 2026-07-11
updated: 2026-07-11
---

# LangGraph 模块详解

本页面是 LangGraph 所有模块的导航中心。LangGraph 采用模块化设计，每个功能领域都是独立的包或子模块，便于按需引入和组合。

## 模块目录树

```
wiki/langgraph/modules/
├── LangGraph模块详解.md              # 本页（导航入口）
├── langgraph_core/                  # 核心图引擎
│   ├── 概述.md
│   ├── StateGraph.md
│   ├── 节点.md
│   ├── 边.md
│   ├── 编译与执行.md
│   ├── 并行执行模型.md              # 超级步骤、同步 vs 异步
│   ├── 错误处理与恢复.md
│   ├── Command与状态更新.md
│   └── 可视化与调试.md
├── langgraph_state/                 # 状态管理
│   ├── 概述.md
│   ├── 状态定义.md
│   ├── Reducer.md
│   └── 状态更新模式.md
├── langgraph_checkpoint/            # 持久化
│   ├── 概述.md
│   ├── InMemorySaver.md
│   ├── PostgresSaver.md
│   ├── SqliteSaver.md
│   └── thread_id与会话管理.md
├── langgraph_streaming/             # 流式输出
│   ├── 概述.md
│   ├── 流式模式对比.md
│   └── 异步流式.md
├── langgraph_human_in_loop/         # 人机交互
│   ├── 概述.md
│   ├── interrupt机制.md
│   └── 审批工作流.md
├── langgraph_multi_agent/           # 多 Agent
│   ├── 概述.md
│   ├── 主管模式.md
│   ├── 网络模式.md
│   └── 层级模式.md
└── langgraph_subgraph/              # 子图
    ├── 概述.md
    └── 子图组合模式.md
```

## 子页面

### langgraph_core — 核心图引擎
- [[langgraph/modules/langgraph_core/概述|概述]]
- [[langgraph/modules/langgraph_core/StateGraph|StateGraph]]
- [[langgraph/modules/langgraph_core/节点|节点]]
- [[langgraph/modules/langgraph_core/边|边]]
- [[langgraph/modules/langgraph_core/编译与执行|编译与执行]]
- [[langgraph/modules/langgraph_core/并行执行模型|并行执行模型]]
- [[langgraph/modules/langgraph_core/错误处理与恢复|错误处理与恢复]]
- [[langgraph/modules/langgraph_core/Command与状态更新|Command 与状态更新]]
- [[langgraph/modules/langgraph_core/可视化与调试|可视化与调试]]

### langgraph_state — 状态管理
- [[langgraph/modules/langgraph_state/概述|概述]]
- [[langgraph/modules/langgraph_state/状态定义|状态定义]]
- [[langgraph/modules/langgraph_state/Reducer|Reducer]]
- [[langgraph/modules/langgraph_state/状态更新模式|状态更新模式]]

### langgraph_checkpoint — 持久化
- [[langgraph/modules/langgraph_checkpoint/概述|概述]]
- [[langgraph/modules/langgraph_checkpoint/InMemorySaver|InMemorySaver]]
- [[langgraph/modules/langgraph_checkpoint/PostgresSaver|PostgresSaver]]
- [[langgraph/modules/langgraph_checkpoint/SqliteSaver|SqliteSaver]]
- [[langgraph/modules/langgraph_checkpoint/thread_id与会话管理|thread_id与会话管理]]

### langgraph_streaming — 流式输出
- [[langgraph/modules/langgraph_streaming/概述|概述]]
- [[langgraph/modules/langgraph_streaming/流式模式对比|流式模式对比]]
- [[langgraph/modules/langgraph_streaming/异步流式|异步流式]]

### langgraph_human_in_loop — 人机交互
- [[langgraph/modules/langgraph_human_in_loop/概述|概述]]
- [[langgraph/modules/langgraph_human_in_loop/interrupt机制|interrupt机制]]
- [[langgraph/modules/langgraph_human_in_loop/审批工作流|审批工作流]]

### langgraph_multi_agent — 多 Agent
- [[langgraph/modules/langgraph_multi_agent/概述|概述]]
- [[langgraph/modules/langgraph_multi_agent/主管模式|主管模式]]
- [[langgraph/modules/langgraph_multi_agent/网络模式|网络模式]]
- [[langgraph/modules/langgraph_multi_agent/层级模式|层级模式]]

### langgraph_subgraph — 子图
- [[langgraph/modules/langgraph_subgraph/概述|概述]]
- [[langgraph/modules/langgraph_subgraph/子图组合模式|子图组合模式]]

### tutorial — 实战教程
- [[langgraph/tutorial/综合示例_全概念演示|综合示例：一个文件串起全部概念]] ⭐ 最小演示盘
- [[langgraph/tutorial/ReAct_Agent实现|ReAct Agent 实现]]
- [[langgraph/tutorial/多轮对话Agent|多轮对话 Agent]]
- [[langgraph/tutorial/RAG_Agent|RAG Agent]]
- [[langgraph/tutorial/测试模式|测试模式]]
- [[langgraph/tutorial/生产部署清单|生产部署清单]]

## 模块分类表

| 模块 | 包名 | 核心职责 | 难度 | 优先级 |
|------|------|---------|------|--------|
| [[langgraph/modules/langgraph_core/概述|langgraph_core]] | `langgraph` | 图引擎：StateGraph、节点、边、编译 | 基础 | 必学 |
| [[langgraph/modules/langgraph_state/概述|langgraph_state]] | `langgraph` | 状态定义、Reducer、状态模式 | 基础 | 必学 |
| [[langgraph/modules/langgraph_checkpoint/概述|langgraph_checkpoint]] | `langgraph-checkpoint-*` | 状态持久化、对话恢复 | 中级 | 推荐 |
| [[langgraph/modules/langgraph_streaming/概述|langgraph_streaming]] | `langgraph` | 流式输出、实时反馈 | 中级 | 推荐 |
| [[langgraph/modules/langgraph_human_in_loop/概述|langgraph_human_in_loop]] | `langgraph` | 中断、审批、人工输入 | 高级 | 进阶 |
| [[langgraph/modules/langgraph_subgraph/概述|langgraph_subgraph]] | `langgraph` | 子图创建、组合、复用 | 高级 | 进阶 |
| [[langgraph/modules/langgraph_multi_agent/概述|langgraph_multi_agent]] | `langgraph` | 多 Agent 协作架构 | 高级 | 进阶 |

## 推荐学习路径

### 路径一：快速上手（1-2 小时）

适合想快速构建简单 Agent 的开发者。

```
1. [[langgraph/overview/LangGraph概述|LangGraph 概述]]        — 了解核心思想
2. [[langgraph/quickstart/快速入门|快速入门]]                  — 动手实践
3. [[langgraph/modules/langgraph_core/概述|核心模块概述]]      — 理解 API
4. [[langgraph/modules/langgraph_core/StateGraph|StateGraph]]  — 掌握图构建
```

### 路径二：系统学习（1-2 天）

适合想深入理解 LangGraph 架构的开发者。

```
1. [[langgraph/overview/LangGraph概述|LangGraph 概述]]
2. [[langgraph/quickstart/快速入门|快速入门]]
3. [[langgraph/modules/langgraph_core/概述|核心模块概述]]
4. [[langgraph/modules/langgraph_core/StateGraph|StateGraph]]
5. [[langgraph/modules/langgraph_core/节点|节点]]
6. [[langgraph/modules/langgraph_core/边|边]]
7. [[langgraph/modules/langgraph_core/编译与执行|编译与执行]]
8. [[langgraph/modules/langgraph_state/概述|状态管理]]
9. [[langgraph/modules/langgraph_checkpoint/概述|持久化]]
10. [[langgraph/modules/langgraph_streaming/概述|流式传输]]
```

### 路径三：高级应用（1 周+）

适合构建生产级多 Agent 系统的开发者。

```
完成路径二后，继续：

11. [[langgraph/modules/langgraph_human_in_loop/概述|人机协作]]
12. [[langgraph/modules/langgraph_subgraph/概述|子图]]
13. [[langgraph/modules/langgraph_multi_agent/概述|多 Agent 系统]]
14. 实战教程（tutorial/）
```

## 模块依赖关系

```
┌─────────────────────────────────────────────────────────┐
│                    你的应用层                             │
├─────────────────────────────────────────────────────────┤
│  multi_agent    human_in_loop    subgraph               │
│       │               │              │                   │
│       └───────────────┼──────────────┘                   │
│                       ▼                                  │
│              streaming    checkpoint                     │
│                       │              │                   │
│                       └──────┬───────┘                   │
│                              ▼                           │
│                        ┌──────────┐                      │
│                        │  state   │                      │
│                        └────┬─────┘                      │
│                             ▼                            │
│                    ┌────────────────┐                    │
│                    │  langgraph_core │                    │
│                    │  (StateGraph,   │                    │
│                    │   Node, Edge)   │                    │
│                    └────────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

**依赖说明**：
- `langgraph_core` 是基础，所有其他模块都依赖它
- `state` 和 `checkpoint` 是中间层，被高级模块使用
- `multi_agent`、`human_in_loop`、`subgraph` 是顶层应用模式

## 快速参考

### 核心 API 速查

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

# 定义状态
class State(TypedDict):
    messages: list

# 构建图
builder = StateGraph(State)
builder.add_node("node_name", node_function)
builder.add_edge(START, "node_name")
builder.add_edge("node_name", END)

# 编译（可选：添加持久化）
graph = builder.compile(checkpointer=MemorySaver())

# 执行
result = graph.invoke({"messages": []})
```

### 常用导入

```python
# 图构建
from langgraph.graph import StateGraph, START, END, MessagesState

# 持久化
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver

# 预构建组件
from langgraph.prebuilt import ToolNode, create_react_agent

# 人机协作
from langgraph.types import interrupt, Command

# 流式传输
# graph.stream(inputs, stream_mode="updates")
```

## 相关页面

- [[langgraph/overview/LangGraph概述|LangGraph 概述]] — 了解 LangGraph 的核心思想
- [[langgraph/quickstart/快速入门|快速入门]] — 从零开始的教程
- [[langchain/overview/LangChain概述|LangChain 概述]] — LangChain 与 LangGraph 的关系
