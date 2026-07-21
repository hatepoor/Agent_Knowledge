---
tags: [langchain, checkpoints, metadata, state]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# Checkpoint 元数据

每个检查点（Checkpoint）包含完整的状态信息和元数据，用于追踪 Agent 的执行历史。

## 元数据结构

| 字段 | 描述 |
|------|------|
| `config` | 配置（thread_id、checkpoint_id） |
| `values` | 状态值（messages、状态字段） |
| `next` | 下一步要执行的节点 |
| `metadata` | 元数据（step、source、writes） |
| `parent_config` | 父检查点（用于时间旅行） |
| `tasks` | 任务列表 |

## 读取检查点

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
config = {"configurable": {"thread_id": "user-1-session"}}

# ... agent 执行若干步骤后 ...

# 读取最新检查点
state = checkpointer.get(config)
print(f"Thread: {state['configurable']['thread_id']}")
print(f"Messages: {len(state.values['messages'])}")
print(f"Step: {state.metadata['step']}")
```

## 读取完整 CheckpointTuple

```python
# 获取包含完整元数据的 tuple
cp_tuple = checkpointer.get_tuple(config)

print(f"Config: {cp_tuple.config}")
print(f"Values: {cp_tuple.values}")
print(f"Metadata: {cp_tuple.metadata}")
print(f"Next: {cp_tuple.next}")
print(f"Parent: {cp_tuple.parent_config}")
```

## 列出所有检查点

```python
# 列出 thread 的所有检查点（按时间倒序）
checkpoints = list(checkpointer.list(config))
for cp in checkpoints:
    print(f"Step {cp.metadata['step'][[}: {cp.config}")
```

## metadata 中的常见字段

| 字段 | 说明 |
|------|------|
| `step` | 当前步骤编号 |
| `source` | 状态来源（`input`、`loop`、`update`） |
| `writes` | 本步骤写入的数据 |
| `parents` | 父节点信息 |

## 注意事项

- Checkpoint 包含**完整状态**，大型对话可能占用较多存储
- `parent_config` 指向父检查点，用于时间旅行时追溯执行路径
- `next` 字段标识 Agent 下一步要执行的节点，用于中断恢复
- `tasks` 列表包含待执行的子任务

## 相关

- [[langchain/modules/langchain_checkpoints/概述|<- 返回 checkpoints 概述]]
- [[langchain/modules/langchain_checkpoints/时间旅行|时间旅行]]
