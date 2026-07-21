---
tags: [langchain, stores, checkpointer, comparison, memory]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# Store 与 Checkpointer 对比

Store 和 Checkpointer 都是 LangGraph/LangChain 的持久化组件，但用途和设计目标完全不同。

## 对比总览

| 维度 | Store | Checkpointer |
|------|-------|--------------|
| 用途 | 长期记忆（事实、偏好） | 短期记忆（消息历史） |
| 数据结构 | 键值对 | 状态快照 |
| 生命周期 | 永久 | 取决于 thread |
| 多用户 | 通过 namespace 隔离 | 通过 thread_id 区分 |
| 检索 | 精确 + 部分语义 | 按 thread_id 取 |

## 核心区别

### Store -- 长期记忆

Store 存储的是**与对话无关的持久化数据**，跨 thread 存在：

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

# 保存用户偏好（在任何 thread 中都可访问）
store.put(("users", "alice"), "favorite_color", {"value": "blue"})

# 在新 thread 中读取（即使 thread_id 不同）
item = store.get(("users", "alice"), "favorite_color")
print(item.value)  # {"value": "blue"}
```

### Checkpointer -- 短期记忆

Checkpointer 保存的是**特定对话 thread 的完整状态**，包括消息历史：

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

# Thread 1 中的对话
config1 = {"configurable": {"thread_id": "session-1"}}
agent.invoke({"messages": [{"role": "user", "content": "Hi, I'm Alice."}]}, config1)

# Thread 2 看不到 Thread 1 的历史
config2 = {"configurable": {"thread_id": "session-2"}}
# agent 不知道用户名字
```

## 何时使用哪个

| 场景 | 推荐 |
|------|------|
| 用户偏好、个人设置 | Store |
| 对话消息历史 | Checkpointer |
| 跨会话持久化知识 | Store |
| 会话内状态恢复 | Checkpointer |
| 时间旅行 | Checkpointer |
| 人机交互中断恢复 | Checkpointer |
| 多用户数据隔离 | Store（namespace） |
| 多会话隔离 | Checkpointer（thread_id） |

## 同时使用

生产环境中通常**两者同时使用**：

```python
from langgraph.store.memory import InMemoryStore
from langgraph.checkpoint.memory import InMemorySaver

store = InMemoryStore()
checkpointer = InMemorySaver()

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[save_user_pref, read_user_pref][[,
    store=store,           # 长期记忆
    checkpointer=checkpointer,  # 短期记忆
)
```

## 注意事项

- Store 数据不受 Checkpointer 的 thread 隔离影响
- Checkpointer 的状态快照可能很大（包含所有 messages）
- 两者使用不同的持久化后端（可相同也可不同）
- 部署 LangGraph Server 时 Checkpointer 自动配置，Store 需手动注入

## 相关

- [[langchain/modules/langchain_stores/概述|<- 返回 stores 概述]]
- [[langchain/modules/langchain_checkpoints/概述|checkpoints 模块]]
