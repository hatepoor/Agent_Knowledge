---
tags: [langchain, checkpoints, thread-id, multi-turn, conversation]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# 多轮对话与 thread_id

Checkpointer 通过 `thread_id` 实现**多轮对话的上下文隔离**，同一 thread_id 下的对话共享消息历史。

## 基本概念

- 每个 `thread_id` 代表一个独立的对话线程
- 同一 thread_id 的多次 `invoke` 共享消息历史
- 不同 thread_id 的对话完全隔离

## 基本用法

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."

from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent

checkpointer = InMemorySaver()

agent = create_agent(
    "openai:gpt-5.5",
    tools=[],
    checkpointer=checkpointer,
)
```

## 同一 thread 多轮对话

```python
config = {"configurable": {"thread_id": "user-1-session"}}

# 第 1 轮
r1 = agent.invoke(
    {"messages": [{"role": "user", "content": "Hi, I'm Alice."}]},
    config,
)

# 第 2 轮 -- Agent 记得第 1 轮的内容
r2 = agent.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    config,
)
print(r2["messages"][-1].content)  # 应回答 "Alice"
```

## 不同 thread 隔离

```python
config2 = {"configurable": {"thread_id": "user-1-other"}}

# 新 thread -- Agent 不知道之前的对话
r3 = agent.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    config2,
)
# 应回答 "I don't know"（新 thread 无历史）
```

## thread_id 设计建议

| 场景 | thread_id 格式 | 说明 |
|------|---------------|------|
| 单用户会话 | `"user-{user_id}-session-{session_id}"` | 每个会话独立 |
| 多用户 | `"user-{user_id}"` | 用户级隔离 |
| 临时对话 | `uuid7()` | 每次新对话 |

```python
from langchain_core.utils.uuid import uuid7

# 生成唯一 thread_id
config = {"configurable": {"thread_id": str(uuid7())}}
```

## 与 Postgres 生产版配合

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://user:pass@localhost:5432/checkpoints"
with PostgresSaver.from_conn_string(DB_URI) as cp:
    cp.setup()  # 自动建表
    agent = create_agent(model, tools=[...], checkpointer=cp)
    # 跨进程、跨机器、跨重启都保留
```

## 与 SQLite 轻量版配合

```python
from langgraph.checkpoint.sqlite import SqliteSaver

with SqliteSaver.from_conn_string("./checkpoints.db") as cp:
    agent = create_agent(model, tools=[...][[, checkpointer=cp)
```

## 注意事项

- `thread_id` 是**唯一**的隔离维度，相同 thread_id 共享上下文
- 建议使用有意义的命名规则（如包含用户ID和会话ID）
- PostgresSaver 和 SqliteSaver 都自动按 thread_id 区分数据
- 删除 thread 使用 `checkpointer.delete_thread(thread_id)`

## 相关

- [[langchain/modules/langchain_checkpoints/概述|<- 返回 checkpoints 概述]]
- [[langchain/modules/langchain_checkpoints/Checkpoint元数据|Checkpoint 元数据]]
