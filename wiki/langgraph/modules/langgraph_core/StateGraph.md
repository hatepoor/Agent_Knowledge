---
tags: [langgraph, core, stategraph, api, graph]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# StateGraph 详解

`StateGraph` 是 LangGraph 的核心类，用于定义和构建状态图。它提供了声明式 API 来注册节点、定义边，并最终编译为可执行的图。

## 构造函数

```python
StateGraph(state_schema, config_schema=None)
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `state_schema` | `Type[TypedDict]` 或 `Type[BaseModel]` | 状态的结构定义 |
| `config_schema` | `Type[TypedDict]` 或 `Type[BaseModel]`，可选 | 配置的结构定义 |

### 示例

```python
from typing import TypedDict
from langgraph.graph import StateGraph

# 方式一：使用 TypedDict（最常用）
class State(TypedDict):
    messages: list
    count: int

builder = StateGraph(State)

# 方式二：使用 Pydantic BaseModel
from pydantic import BaseModel

class State(BaseModel):
    messages: list = []
    count: int = 0

builder = StateGraph(State)

# 方式三：使用内置的 MessagesState（聊天机器人专用）
from langgraph.graph import MessagesState

builder = StateGraph(MessagesState)
```

## 状态模式（State Schema）

### TypedDict（推荐）

最轻量、最常用的方式：

```python
from typing import TypedDict, Annotated
from langgraph.graph import add_messages
from langchain_core.messages import BaseMessage

class State(TypedDict):
    """自定义状态结构"""
    # 使用 Annotated 和 reducer 控制更新行为
    messages: Annotated[list[BaseMessage], add_messages]
    
    # 普通字段（新值覆盖旧值）
    user_name: str
    step_count: int
    
    # 带默认值的字段
    retry_count: int = 0
```

### MessagesState（聊天专用）

内置的消息状态，预定义了 `messages` 字段和 `add_messages` reducer：

```python
from langgraph.graph import MessagesState

# MessagesState 等价于：
# class MessagesState(TypedDict):
#     messages: Annotated[list[BaseMessage], add_messages]

builder = StateGraph(MessagesState)
```

### Pydantic BaseModel

适合需要数据验证的复杂状态：

```python
from pydantic import BaseModel, Field
from typing import List
from langchain_core.messages import BaseMessage

class State(BaseModel):
    messages: List[BaseMessage] = Field(default_factory=list)
    user_name: str = Field(default="anonymous")
    max_retries: int = Field(default=3, ge=0, le=10)
    
    class Config:
        arbitrary_types_allowed = True  # 允许 BaseMessage 类型

builder = StateGraph(State)
```

## 核心方法

### add_node()

添加节点到图。

```python
add_node(name, action, metadata=None, input=None, retry=None)
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | `str` | 节点名称（唯一标识） |
| `action` | `Callable` 或 `CompiledGraph` | 节点逻辑（函数、类或子图） |
| `metadata` | `dict`，可选 | 节点元数据 |
| `input` | `Type[TypedDict]`，可选 | 节点的输入状态模式 |
| `retry` | `RetryPolicy`，可选 | 重试策略 |

#### 示例

```python
# 方式一：添加函数节点
def my_node(state: State) -> dict:
    return {"count": state["count"] + 1}

builder.add_node("increment", my_node)

# 方式二：添加异步函数节点
async def async_node(state: State) -> dict:
    result = await some_async_operation()
    return {"result": result}

builder.add_node("async_task", async_node)

# 方式三：添加子图作为节点
subgraph = StateGraph(SubState)
# ... 构建子图 ...
builder.add_node("subtask", subgraph.compile())
```

### add_edge()

添加普通边（固定连接）。

```python
add_edge(from_node, to_node)
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `from_node` | `str` 或 `START` | 起始节点 |
| `to_node` | `str` 或 `END` | 目标节点 |

#### 示例

```python
from langgraph.graph import START, END

# 入口边
builder.add_edge(START, "first_node")

# 节点间边
builder.add_edge("node_a", "node_b")

# 出口边
builder.add_edge("last_node", END)
```

### add_conditional_edges()

添加条件边（动态路由）。

```python
add_conditional_edges(from_node, path, path_map=None)
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `from_node` | `str` | 起始节点 |
| `path` | `Callable` | 路由函数，接收状态，返回下一个节点名 |
| `path_map` | `dict`，可选 | 节点名映射（可选，用于重命名） |

#### 示例

```python
from typing import Literal

# 路由函数
def route_next(state: State) -> Literal["node_a", "node_b", "__end__"]:
    """根据状态决定下一个节点"""
    if state["should_stop"]:
        return "__end__"
    elif state["condition"] == "a":
        return "node_a"
    else:
        return "node_b"

# 添加条件边
builder.add_conditional_edges("decision_node", route_next)
```

### compile()

编译图，生成可执行的 `CompiledStateGraph`。

```python
compile(checkpointer=None, interrupt_before=None, interrupt_after=None)
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `checkpointer` | `BaseCheckpointSaver`，可选 | 状态持久化器 |
| `interrupt_before` | `list[str]`，可选 | 在这些节点执行前中断 |
| `interrupt_after` | `list[str]`，可选 | 在这些节点执行后中断 |

#### 示例

```python
from langgraph.checkpoint.memory import MemorySaver

# 基础编译
graph = builder.compile()

# 带持久化编译
checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# 带中断编译（人机协作）
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["sensitive_action"]
)
```

## 完整示例：多节点工作流

```python
"""
完整示例：构建一个多节点的内容处理工作流
"""
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI

# 1. 定义状态
class ContentState(TypedDict):
    """内容处理状态"""
    raw_content: str      # 原始内容
    summary: str          # 摘要
    keywords: list        # 关键词
    translated: str       # 翻译结果
    quality_score: float  # 质量评分

# 2. 初始化 LLM
model = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# 3. 定义节点
def summarize(state: ContentState) -> dict:
    """生成摘要"""
    prompt = f"请用一句话总结以下内容：\n{state['raw_content']}"
    response = model.invoke(prompt)
    return {"summary": response.content}

def extract_keywords(state: ContentState) -> dict:
    """提取关键词"""
    prompt = f"请提取以下内容的3-5个关键词，用逗号分隔：\n{state['raw_content']}"
    response = model.invoke(prompt)
    keywords = [k.strip() for k in response.content.split(",")]
    return {"keywords": keywords}

def translate(state: ContentState) -> dict:
    """翻译为英文"""
    prompt = f"请将以下内容翻译为英文：\n{state['raw_content']}"
    response = model.invoke(prompt)
    return {"translated": response.content}

def assess_quality(state: ContentState) -> dict:
    """评估质量"""
    # 简单规则：有摘要、关键词和翻译，质量就高
    score = 0.0
    if state.get("summary"):
        score += 0.3
    if state.get("keywords"):
        score += 0.3
    if state.get("translated"):
        score += 0.4
    return {"quality_score": score}

def route_by_quality(state: ContentState) -> Literal["output", "retry"]:
    """根据质量决定下一步"""
    if state["quality_score"] >= 0.8:
        return "output"
    else:
        return "retry"

def output_result(state: ContentState) -> dict:
    """输出结果"""
    print(f"✓ 处理完成，质量评分: {state['quality_score']}")
    return {}

def retry_processing(state: ContentState) -> dict:
    """重试处理"""
    print(f"⚠ 质量不足，重新处理...")
    return {}

# 4. 构建图
builder = StateGraph(ContentState)

# 添加节点
builder.add_node("summarize", summarize)
builder.add_node("keywords", extract_keywords)
builder.add_node("translate", translate)
builder.add_node("assess", assess_quality)
builder.add_node("output", output_result)
builder.add_node("retry", retry_processing)

# 添加边
builder.add_edge(START, "summarize")
builder.add_edge("summarize", "keywords")
builder.add_edge("keywords", "translate")
builder.add_edge("translate", "assess")

# 条件边：根据质量决定输出或重试
builder.add_conditional_edges("assess", route_by_quality)
builder.add_edge("output", END)
builder.add_edge("retry", "summarize")  # 重试回到摘要步骤

# 5. 编译
graph = builder.compile()

# 6. 执行
result = graph.invoke({
    "raw_content": """
    LangGraph 是一个用于构建有状态 AI Agent 的低级编排框架。
    它通过图结构来定义工作流，支持循环、条件分支和持久化。
    LangGraph 与 LangChain 互补，LangChain 提供组件，LangGraph 提供编排。
    """
})

print(f"\n最终结果:")
print(f"摘要: {result['summary']}")
print(f"关键词: {result['keywords']}")
print(f"翻译: {result['translated'][:100]}...")
print(f"质量评分: {result['quality_score']}")
```

## 常见模式

### 模式一：简单管道

```python
builder.add_edge(START, "A")
builder.add_edge("A", "B")
builder.add_edge("B", "C")
builder.add_edge("C", END)
```

### 模式二：并行分支（Fan-out）

```python
# A 执行后，B 和 C 并行执行
builder.add_edge(START, "A")
builder.add_edge("A", "B")
builder.add_edge("A", "C")
builder.add_edge("B", "D")
builder.add_edge("C", "D")
builder.add_edge("D", END)
```

### 模式三：条件循环

```python
def should_retry(state):
    if state["retry_count"] < 3 and state["success"] == False:
        return "retry"
    return "end"

builder.add_edge(START, "process")
builder.add_conditional_edges("process", should_retry)
builder.add_edge("retry", "process")
builder.add_edge("end", END)
```

## 最佳实践

1. **状态设计要精简**：只包含必要的字段，避免在状态中存储大量数据
2. **节点职责要单一**：每个节点只做一件事，便于测试和复用
3. **使用类型提示**：为状态和节点函数添加类型提示，提高代码可读性
4. **合理使用条件边**：避免过于复杂的路由逻辑，保持图的可读性
5. **考虑使用子图**：当图变得复杂时，拆分为子图提高可维护性

## 相关页面

- [[langgraph/modules/langgraph_core/概述|← 返回核心模块概述]]
- [[langgraph/modules/langgraph_core/节点|节点详解]] — 节点的类型和用法
- [[langgraph/modules/langgraph_core/边|边详解]] — 边的类型和路由
- [[langgraph/modules/langgraph_core/编译与执行|编译与执行]] — 编译选项和执行模式
