---
tags: [langchain, messages, RemoveMessage, history-trimming, reducer]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# RemoveMessage 历史裁剪

## 概述

`RemoveMessage` 是 LangChain 中用于**删除消息历史**的特殊消息类型。它配合 LangGraph 的消息 reducer 使用，可以实现对话历史的裁剪，防止 token 溢出或控制上下文窗口大小。

## 基本用法

```python
from langchain.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES

# 删除所有消息
delete_all = [RemoveMessage(id=REMOVE_ALL_MESSAGES)]

# 删除指定消息
delete_one = [RemoveMessage(id="msg_123")]
```

## 删除所有消息

当需要清空整个对话历史时：

```python
from langchain.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES

# 在状态更新中返回 RemoveMessage
def clear_history(state):
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}
```

## 删除指定消息

当需要删除特定消息（如过期的工具调用结果）时：

```python
from langchain.messages import RemoveMessage

def trim_old_messages(state):
    messages = state["messages"]
    # 保留最近的 10 条，删除其余
    to_remove = messages[:-10]
    return {
        "messages": [RemoveMessage(id=msg.id) for msg in to_remove]
    }
```

## 在 LangGraph 中使用

`RemoveMessage` 需要配合消息 reducer（`add_messages`）使用。LangGraph 的状态图在合并消息时会自动识别 `RemoveMessage` 并执行删除：

```python
from langgraph.graph import StateGraph
from langgraph.graph.message import add_messages
from typing import Annotated
from langchain.agents import AgentState

class MyState(AgentState):
    messages: Annotated[list, add_messages][[  # 启用消息合并（含删除）

graph = StateGraph(MyState)
# ... 添加节点和边
```

## 常见裁剪策略

| 策略 | 说明 |
|------|------|
| 保留最近 N 条 | 固定窗口，删除最旧的消息 |
| 按 token 数裁剪 | 当总 token 超过阈值时裁剪 |
| 保留 SystemMessage | 裁剪时跳过系统消息 |
| 清空全部 | 重置对话状态 |

## 注意事项

- `RemoveMessage` 不会立即生效，而是在状态合并（reducer）时执行
- 删除消息时注意保留必要的上下文（如 SystemMessage）
- `REMOVE_ALL_MESSAGES` 会清空所有消息，包括系统提示词
- 建议配合 `checkpointer` 使用，以便裁剪操作可以被持久化

## 相关页面

- [[langchain/modules/langchain_messages/概述|← 返回 messages 概述]]
- [[langchain/modules/langchain_messages/消息类型详解|消息类型详解]]
- [[langchain/modules/langchain_checkpoints/概述|checkpoints 模块]] — 状态持久化
