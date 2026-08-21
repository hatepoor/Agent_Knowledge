---
tags: [langgraph, state, reducer, 归约函数, add_messages, operator.add, RemoveMessage, Overwrite]
type: concept
created: 2026-07-11
updated: 2026-08-21
---

# Reducer 归约函数

## 什么是 Reducer

**Reducer（归约函数）** 是 LangGraph 状态管理的核心机制。它决定了当一个节点返回某个字段的更新时，如何将这个更新**合并**到现有状态中。

简单来说：

```
当前状态: {"messages": [msg1, msg2], "count": 5}
节点返回: {"messages": [msg3], "count": 6}

使用默认 Reducer（覆盖）: {"messages": [msg3], "count": 6}
使用 add_messages Reducer: {"messages": [msg1, msg2, msg3], "count": 6}
```

**关键点**：每个字段可以有自己的 Reducer。不同字段可以使用不同的合并策略。

---

## 默认行为：覆盖（Overwrite）

如果不为字段指定 Reducer，LangGraph 默认使用**覆盖**策略——节点返回的值直接替换状态中的旧值。

```python
from typing import TypedDict

class SimpleState(TypedDict):
    """没有指定 Reducer 的状态"""
    name: str       # 默认覆盖
    count: int      # 默认覆盖
    data: dict      # 默认覆盖

def my_node(state: SimpleState) -> dict:
    return {
        "name": "新名字",   # 直接替换旧名字
        "count": 10,        # 直接替换旧计数
        # 不返回 data 字段 → data 保持不变
    }
```

### 覆盖行为的特点

- 简单直观：新值 = 最终值
- 适合标量字段（字符串、数字、布尔值）
- 不适合需要累积的字段（如消息列表）

---

## 内置 Reducer：add_messages

`add_messages` 是 LangGraph 最常用的内置 Reducer，专门用于处理消息列表。

### 功能

1. **追加**：将新消息追加到现有消息列表
2. **去重**：如果新消息的 ID 与现有消息相同，则替换（更新）该消息
3. **排序**：保持消息的时间顺序

### 导入与使用

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages
from langchain_core.messages import HumanMessage, AIMessage

# 方式一：使用 Annotated 类型标注（推荐）
class ChatState(TypedDict):
    messages: Annotated[list, add_messages]  # 使用 add_messages 归约
    query: str                                # 默认覆盖

# 方式二：在 StateGraph 构造时指定（不常用）
# graph = StateGraph(dict)
# graph.add_channel("messages", add_messages)
```

### add_messages 行为详解

```python
from langgraph.graph.message import add_messages
from langchain_core.messages import HumanMessage, AIMessage

# 场景1：追加新消息
existing = [HumanMessage(content="你好", id="msg-1")]
new = [AIMessage(content="你好！有什么可以帮你的？", id="msg-2")]
result = add_messages(existing, new)
# result = [
#     HumanMessage(content="你好", id="msg-1"),
#     AIMessage(content="你好！有什么可以帮你的？", id="msg-2"),
# ]

# 场景2：更新已有消息（相同 ID → 替换）
existing = [
    HumanMessage(content="你好", id="msg-1"),
    AIMessage(content="旧回答", id="msg-2"),
]
update = [AIMessage(content="新回答（已修正）", id="msg-2")]
result = add_messages(existing, update)
# result = [
#     HumanMessage(content="你好", id="msg-1"),
#     AIMessage(content="新回答（已修正）", id="msg-2"),  # 被替换了
# ]
```

---

## 其他内置 Reducer

除 `add_messages` 外，LangGraph 还提供 3 个常用内置机制：`operator.add`（通用累加）、`RemoveMessage`（删除消息）、`Overwrite`（强制覆盖）。加上默认的覆盖行为，构成完整的 reducer 工具集。

### 1. operator.add — 通用累加（数值 / 列表）

来自 Python 标准库 `operator`，不需要 LangGraph 额外实现——任何 `(left, right) -> left + right` 的二元函数都能当 reducer，`operator.add` 是最标准的那个。

```python
from operator import add
from typing import TypedDict, Annotated

class AccumulateState(TypedDict):
    total_score: Annotated[int, add]        # 数值累加：每次返回的分数累加
    results: Annotated[list[str], add]      # 列表拼接：收集多个节点的结果
    name: str                                # 默认覆盖

def node_a(state: AccumulateState) -> dict:
    return {"total_score": 5, "results": ["节点A的结果"]}

def node_b(state: AccumulateState) -> dict:
    return {"total_score": 3, "results": ["节点B的结果"]}

# 初始 {"total_score": 0, "results": ["初始"]}
# 运行后 → {"total_score": 8, "results": ["初始", "节点A的结果", "节点B的结果"]}
```

**vs `add_messages` 的区别**：

| 维度 | `operator.add` | `add_messages` |
|------|---------------|----------------|
| 列表行为 | 纯拼接 | 追加 + 按 ID 更新 + 反序列化 |
| 能更新已有元素 | ❌ | ✅（同 ID 替换） |
| 能删除元素 | ❌ | ✅（配 RemoveMessage） |
| 适用 | 数值计数、非消息列表收集 | 消息历史 |

> ⚠️ 处理**消息列表必须用 `add_messages`**：若用 `operator.add`，HITL 人工修改消息、按 ID 去重等能力全部失效。

---

### 2. RemoveMessage — 删除消息（配 add_messages 使用）

LangGraph 中删除消息的专用机制。`RemoveMessage` 是一个特殊消息对象，放进节点返回的 `messages` 字段，`add_messages` reducer 识别后**按 ID 删除**对应消息。

```python
from langchain_core.messages import RemoveMessage

def delete_old_messages(state):
    """消息超过 6 条时，删除最早的 6 条"""
    messages = state["messages"]
    if len(messages) > 6:
        # 取出最早 6 条消息的 ID，包装成 RemoveMessage 返回
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:6]]}
    return {"messages": []}
```

**清空全部消息**——使用 `REMOVE_ALL_MESSAGES` 常量：

```python
from langgraph.graph.message import REMOVE_ALL_MESSAGES

def clear_all(state):
    """一键清空对话历史"""
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}
```

**关键前提**：
- `RemoveMessage` 只能作用于**使用 `add_messages` reducer 的消息字段**（普通列表字段无法删除）
- 删除必须通过**节点返回**实现——单独调用方法不会生效
- 删除后消息不可恢复，需确保剩余对话仍然语义完整

**典型应用**：
- 长对话裁剪（保留最近 N 条）
- HITL 中删除被拒绝的 AI 回复
- 清除敏感/过期消息

---

### 3. Overwrite — 强制覆盖（绕过 reducer）

当字段配置了 reducer（如 `operator.add` 累加），但某次更新**不想累加、要整体重置**时，用 `Overwrite` 包裹新值，告诉 LangGraph"这次绕过 reducer，直接覆盖"。

```python
from langgraph.types import Overwrite

def reset_messages(state):
    """重置消息列表（绕过 add_messages 的追加逻辑）"""
    return {"messages": Overwrite(["fresh start"])}

def reset_total(state):
    """重置计数器（绕过 operator.add 的累加逻辑）"""
    return {"total_score": Overwrite(0)}
```

**行为对比**：

```python
# 无 Overwrite：走 reducer
return {"total_score": 3}
# → total_score 累加：旧值 + 3

# 有 Overwrite：绕过 reducer
return {"total_score": Overwrite(0)}
# → total_score 直接变为 0，忽略旧值
```

**适用场景**：累计类字段的"清零/重置"操作——比如一轮任务结束后重置计数器、清空临时收集的结果列表。

---

### 4. 内置 reducer 速查总表

| Reducer | 导入来源 | 作用 | 适用字段 |
|---------|---------|------|---------|
| `add_messages` | `langgraph.graph.message` | 追加 + 按 ID 更新 + 反序列化 + 可删除 | 消息历史 |
| `operator.add` | `operator` | 数值累加 / 列表拼接 | 计数、结果收集 |
| `RemoveMessage` | `langchain_core.messages` | 按 ID 删除消息（配 add_messages） | 消息历史 |
| `Overwrite` | `langgraph.types` | 绕过 reducer 强制覆盖 | 任何配了 reducer 的字段 |
| （默认）无 | — | 新值替换旧值 | 标量字段 |

> 补充：`MessagesState` 是预置状态类（`langgraph.graph` 导入），内置 `messages` 字段 + `add_messages` reducer，聊天 Agent 可直接继承它扩展。

---

## 自定义 Reducer

当内置 Reducer 不能满足需求时，可以编写自定义 Reducer 函数。

### Reducer 函数签名

```python
def my_reducer(existing_value: Any, new_value: Any) -> Any:
    """
    自定义 Reducer 函数

    参数:
        existing_value: 状态中该字段的当前值
        new_value: 节点返回的该字段的新值

    返回:
        合并后的最终值
    """
    ...
```

### 示例1：列表追加（不去重）

```python
from typing import TypedDict, Annotated

def append_list(existing: list, new: list) -> list:
    """简单的列表追加 Reducer"""
    return existing + new

class State(TypedDict):
    # 每次节点返回 {"items": [new_item]}，会追加到列表
    items: Annotated[list, append_list]
    name: str  # 默认覆盖
```

### 示例2：数值累加

```python
from typing import TypedDict, Annotated

def add_numbers(existing: int, new: int) -> int:
    """数值累加 Reducer"""
    return existing + new

class CounterState(TypedDict):
    total: Annotated[int, add_numbers]  # 累加而不是覆盖
    label: str                           # 默认覆盖

# 使用示例
def process_node(state: CounterState) -> dict:
    return {"total": 1}  # 每次调用 total += 1

# 调用3次后：state["total"] == 3（而不是 1）
```

### 示例3：字典深度合并

```python
from typing import TypedDict, Annotated

def merge_dicts(existing: dict, new: dict) -> dict:
    """字典深度合并 Reducer"""
    result = {**existing}
    for key, value in new.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = merge_dicts(result[key], value)
        else:
            result[key] = value
    return result

class ConfigState(TypedDict):
    config: Annotated[dict, merge_dicts]  # 深度合并

# 使用示例
def update_config_node(state: ConfigState) -> dict:
    return {
        "config": {
            "llm": {"temperature": 0.5},  # 只更新 temperature
            # 其他 config 字段保持不变
        }
    }
```

### 示例4：取最大值

```python
from typing import TypedDict, Annotated

def max_value(existing: float, new: float) -> float:
    """取最大值 Reducer"""
    return max(existing, new)

class ScoreState(TypedDict):
    best_score: Annotated[float, max_value]  # 只保留最高分

def evaluate_node(state: ScoreState) -> dict:
    new_score = 0.85
    return {"best_score": new_score}  # 只有当 new_score > 当前最高分时才更新
```

### 示例5：最大长度列表（滑动窗口）

```python
from typing import TypedDict, Annotated

def max_length_list(existing: list, new: list, max_len: int = 100) -> list:
    """保持最大长度的列表 Reducer（保留最新的元素）"""
    combined = existing + new
    if len(combined) > max_len:
        return combined[-max_len:]  # 保留最新的 max_len 个元素
    return combined

class SlidingWindowState(TypedDict):
    recent_items: Annotated[list, max_length_list]  # 最多保留100个
```

---

## Annotated 类型标注详解

LangGraph 使用 Python 的 `typing.Annotated` 来将 Reducer 绑定到特定字段。

### 语法

```python
from typing import Annotated

# 格式：字段名: Annotated[类型, Reducer函数]
class MyState(TypedDict):
    messages: Annotated[list, add_messages]  # list 类型 + add_messages Reducer
    count: int                                # int 类型 + 默认覆盖 Reducer
```

### 多个 Annotated 标注

```python
# 可以同时使用多个标注（从左到右依次处理）
from typing import Annotated

class MultiReducerState(TypedDict):
    # 先追加，再限制长度
    items: Annotated[list, append_list, max_length_list]
```

### 常见错误

```python
# ❌ 错误：忘记 Annotated，直接写类型
class WrongState(TypedDict):
    messages: list  # 这会使用默认覆盖，而不是追加！

# ✅ 正确：使用 Annotated 指定 Reducer
class CorrectState(TypedDict):
    messages: Annotated[list, add_messages]  # 使用 add_messages 追加
```

---

## Reducer 执行流程

```
节点执行完毕
    ↓
返回部分更新 dict: {"field_a": new_a, "field_b": new_b}
    ↓
对每个字段，查找其 Reducer
    ↓
field_a: 有自定义 Reducer → reducer_a(existing_a, new_a) → merged_a
field_b: 无自定义 Reducer → 直接覆盖 → merged_b
    ↓
合并到状态: {**current_state, "field_a": merged_a, "field_b": merged_b}
    ↓
传递给下一个节点
```

---

## 完整实战示例

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_core.messages import HumanMessage, AIMessage

# ===== 自定义 Reducer =====
def merge_tags(existing: list, new: list) -> list:
    """合并标签，去重"""
    return list(set(existing + new))

# ===== 定义状态 =====
class AppState(TypedDict):
    messages: Annotated[list, add_messages]  # 消息追加
    tags: Annotated[list, merge_tags]         # 标签合并去重
    query: str                                # 覆盖
    step_count: int                           # 覆盖

# ===== 定义节点 =====
def node_a(state: AppState) -> dict:
    """节点A：添加消息和标签"""
    return {
        "messages": [HumanMessage(content="来自节点A的消息")],
        "tags": ["node-a", "processed"],
        "step_count": 1,
    }

def node_b(state: AppState) -> dict:
    """节点B：添加消息和标签"""
    return {
        "messages": [AIMessage(content="来自节点B的回复")],
        "tags": ["node-b", "reviewed"],
        "query": "更新后的查询",
    }

# ===== 构建图 =====
graph = StateGraph(AppState)
graph.add_node("node_a", node_a)
graph.add_node("node_b", node_b)

graph.add_edge(START, "node_a")
graph.add_edge("node_a", "node_b")
graph.add_edge("node_b", END)

app = graph.compile()

# ===== 运行 =====
result = app.invoke({
    "messages": [],
    "tags": ["initial"],
    "query": "初始查询",
    "step_count": 0,
})

print("消息:", [m.content for m in result["messages"]])
# → ["来自节点A的消息", "来自节点B的回复"]  （追加）

print("标签:", result["tags"])
# → ["initial", "node-a", "processed", "node-b", "reviewed"]  （合并去重）

print("查询:", result["query"])
# → "更新后的查询"  （覆盖）

print("步骤数:", result["step_count"])
# → 1  （覆盖，不是累加）
```

---

## 相关页面

- [[langgraph/modules/langgraph_state/概述|状态管理概述]]
- [[langgraph/modules/langgraph_state/状态定义|状态定义]]
- [[langgraph/modules/langgraph_state/状态更新模式|状态更新模式]]
