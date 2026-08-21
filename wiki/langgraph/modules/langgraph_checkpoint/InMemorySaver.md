---
tags: [langgraph, checkpoint, InMemorySaver, MemorySaver, 内存存储, 开发测试]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# InMemorySaver（内存检查点器）

## 概述

`InMemorySaver`（也称为 `MemorySaver`）是 LangGraph 最简单的 Checkpointer 实现。它将所有检查点存储在进程的内存字典中，适合开发和测试场景。

### 特点

- **零配置**：无需安装任何外部依赖
- **极快**：纯内存操作，无 I/O 开销
- **临时性**：进程重启后所有数据丢失
- **单进程**：不支持跨进程或跨机器共享

### 适用场景

- 本地开发和调试
- 单元测试
- 快速原型验证
- 演示和教程

### 不适用场景

- 生产环境（数据会丢失）
- 多进程/多实例部署
- 需要持久化存储的场景

---

## 导入与创建

```python
# 方式一：从 langgraph.checkpoint.memory 导入（推荐）
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()

# 方式二：旧版本中的名称（等价）
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
```

> **注意**：`MemorySaver` 是较新版本的推荐名称，`InMemorySaver` 是旧名称，两者功能完全相同。

---

## 基本用法

### 最简单的多轮对话

```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.checkpoint.memory import MemorySaver

# 定义一个简单的回显节点
def echo_bot(state: MessagesState) -> dict:
    """回显机器人：重复用户的消息"""
    last_message = state["messages"][-1]
    return {
        "messages": [("assistant", f"你说的是：{last_message.content}")]
    }

# 构建图
graph = StateGraph(MessagesState)
graph.add_node("echo", echo_bot)
graph.add_edge(START, "echo")
graph.add_edge("echo", END)

# 创建内存 Checkpointer
checkpointer = MemorySaver()

# 编译图（传入 checkpointer）
app = graph.compile(checkpointer=checkpointer)

# 第一轮对话
config = {"configurable": {"thread_id": "test-1"}}
result = app.invoke(
    {"messages": [("user", "你好")]},
    config=config
)
print(result["messages"])
# → [HumanMessage("你好"), AIMessage("你说的是：你好")]

# 第二轮对话（同一个 thread_id）
result = app.invoke(
    {"messages": [("user", "世界")]},
    config=config
)
print(result["messages"])
# → [HumanMessage("你好"), AIMessage("你说的是：你好"),
#     HumanMessage("世界"), AIMessage("你说的是：世界")]
# 可以看到第一轮的消息还在！
```

---

## 内部存储结构

`MemorySaver` 使用 Python 字典来存储检查点：

```python
# 内部结构（概念性）
checkpointer._checkpoints = {
    ("thread-1", checkpoint_id_1): {
        "values": {"messages": [...], "query": "..."},
        "metadata": {"source": "loop", "step": 0, "writes": {...}},
        "parent_id": None,
    },
    ("thread-1", checkpoint_id_2): {
        "values": {"messages": [...], "query": "..."},
        "metadata": {"source": "loop", "step": 1, "writes": {...}},
        "parent_id": checkpoint_id_1,
    },
    ("thread-2", checkpoint_id_3): {
        "values": {"messages": [...]},
        "metadata": {"source": "loop", "step": 0, "writes": {...}},
        "parent_id": None,
    },
}
```

### 存储键的组成

每个检查点通过以下键唯一标识：

- `thread_id`：会话标识
- `checkpoint_id`：检查点唯一 ID（UUID）
- `checkpoint_ns`：检查点命名空间（用于子图）

---

## 内存使用估算

### 单条检查点的大小

```
一条检查点 ≈ 状态数据的序列化大小 + 元数据开销

典型场景：
- 简单状态（几个字符串）：~1-5 KB
- 聊天状态（10条消息）：~5-20 KB
- 复杂状态（大量文档）：~50-500 KB
```

### 内存使用估算公式

```
总内存 ≈ 平均检查点大小 × 每轮节点数 × 对话轮数 × 并发对话数

示例：
- 平均检查点大小：10 KB
- 每轮节点数：3
- 对话轮数：20
- 并发对话数：100

总内存 ≈ 10KB × 3 × 20 × 100 = 60 MB
```

### 内存优化建议

```python
# 1. 定期清理不需要的 thread
# MemorySaver 没有内置清理方法，但可以手动管理
# 在测试中，使用新的 MemorySaver 实例隔离测试

# 2. 控制消息列表长度
def trim_messages(state: MessagesState) -> dict:
    """限制消息数量，减少内存占用"""
    messages = state["messages"]
    if len(messages) > 50:
        from langchain_core.messages import RemoveMessage
        to_remove = messages[:-50]
        return {"messages": [RemoveMessage(id=m.id) for m in to_remove]}
    return {}

# 3. 避免在状态中存储大量数据
# ❌ 不推荐：在状态中存储完整文档
class BadState(TypedDict):
    documents: list[dict]  # 可能非常大

# ✅ 推荐：只存储引用或摘要
class GoodState(TypedDict):
    doc_ids: list[str]      # 只存储 ID
    doc_summary: str        # 存储摘要
```

---

## 测试中的使用

### 单元测试示例

```python
import pytest
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.checkpoint.memory import MemorySaver

def test_chatbot_multi_turn():
    """测试多轮对话"""
    # 每个测试使用独立的 MemorySaver
    checkpointer = MemorySaver()

    # 构建图...
    app = graph.compile(checkpointer=checkpointer)
    config = {"configurable": {"thread_id": "test"}}

    # 第一轮
    result1 = app.invoke(
        {"messages": [("user", "你好")]},
        config=config
    )
    assert len(result1["messages"]) == 2

    # 第二轮
    result2 = app.invoke(
        {"messages": [("user", "再见")]},
        config=config
    )
    assert len(result2["messages"]) == 4  # 包含两轮对话

def test_isolated_conversations():
    """测试不同 thread_id 的隔离性"""
    checkpointer = MemorySaver()
    app = graph.compile(checkpointer=checkpointer)

    # 对话A
    config_a = {"configurable": {"thread_id": "conv-a"}}
    app.invoke({"messages": [("user", "A的消息")]}, config=config_a)

    # 对话B
    config_b = {"configurable": {"thread_id": "conv-b"}}
    app.invoke({"messages": [("user", "B的消息")]}, config=config_b)

    # 验证隔离：对话A看不到对话B的消息
    state_a = app.get_state(config_a)
    assert len(state_a.values["messages"]) == 2  # 只有A的消息
```

### Fixture 复用

```python
# conftest.py
import pytest
from langgraph.checkpoint.memory import MemorySaver

@pytest.fixture
def checkpointer():
    """每个测试使用独立的 MemorySaver"""
    return MemorySaver()

@pytest.fixture
def app(checkpointer):
    """编译好的图应用"""
    graph = build_my_graph()
    return graph.compile(checkpointer=checkpointer)
```

---

## 查看检查点数据

```python
# 获取当前状态
state = app.get_state(config)
print(f"当前值: {state.values}")
print(f"下一步: {state.next}")
print(f"检查点ID: {state.config}")

# 获取状态历史
for snapshot in app.get_state_history(config):
    print(f"步骤: {snapshot.metadata.get('step', 'N/A')}")
    print(f"值: {snapshot.values}")
    print("---")
```

---

## 局限性

| 局限 | 说明 | 解决方案 |
|------|------|----------|
| 不持久化 | 进程重启数据丢失 | 使用 PostgresSaver 或 SqliteSaver |
| 单进程 | 不能跨进程共享 | 使用 PostgresSaver |
| 内存限制 | 大量数据会耗尽内存 | 控制消息长度、定期清理 |
| 无事务 | 不支持数据库事务 | 生产环境使用 PostgresSaver |
| 无备份 | 无法备份和恢复 | 使用外部存储方案 |

---

## 相关页面

- [[langgraph/modules/langgraph_checkpoint/概述|持久化概述]]
- [[langgraph/modules/langgraph_checkpoint/PostgresSaver|PostgresSaver]]
- [[langgraph/modules/langgraph_checkpoint/SqliteSaver|SqliteSaver]]
- [[langgraph/modules/langgraph_checkpoint/thread_id与会话管理|thread_id 与会话管理]]
