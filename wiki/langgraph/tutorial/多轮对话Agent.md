---
tags: [langgraph, 多轮对话, checkpointer, memory, 教程, 消息裁剪]
type: tutorial
created: 2026-07-11
updated: 2026-07-11
---

# 多轮对话 Agent

## 概述

在 [[langgraph/tutorial/ReAct_Agent实现|ReAct Agent]] 的基础上，我们添加**记忆（Memory）**功能，使 Agent 能够进行多轮对话。核心机制是 LangGraph 的 **Checkpointer（检查点）**——它会在每个节点执行后自动保存状态快照，使得下一次调用时可以恢复之前的对话上下文。

```
对话线程（thread_id = "user_123"）
┌──────────────────────────────────────────────────┐
│                                                  │
│  第1轮: 用户 "我叫小明"                            │
│         → 状态保存: messages=[用户, AI]            │
│                                                  │
│  第2轮: 用户 "我叫什么名字？"                       │
│         → 恢复状态 → messages 追加                 │
│         → AI: "你叫小明"                           │
│         → 状态保存: messages=[用户, AI, 用户, AI]   │
│                                                  │
│  第3轮: 用户 "我喜欢吃什么？"                       │
│         → 恢复状态 → AI: "你没告诉过我..."          │
│                                                  │
└──────────────────────────────────────────────────┘
```

## 前置准备

```bash
pip install langgraph langchain-openai
```

## 第一步：使用 MessagesState

LangGraph 内置了 `MessagesState`，专门为聊天场景设计：

```python
from langgraph.graph import MessagesState

# MessagesState 等价于：
# class MessagesState(TypedDict):
#     messages: Annotated[list, add_messages]
# 
# 它自动使用 add_messages 作为 Reducer，
# 新消息会自动追加到列表中
```

> `MessagesState` 是一个便捷的预定义状态，内部已经配置好了 `add_messages` Reducer。详见 [[langgraph/modules/langgraph_state/概述|状态管理概述]]。

## 第二步：添加 Checkpointer

Checkpointer 是多轮对话的核心。它负责：
- 每次节点执行后**保存状态快照**
- 下次调用时根据 `thread_id` **恢复状态**

```python
from langgraph.checkpoint.memory import MemorySaver

# 内存检查点保存器（开发/测试用）
# 生产环境应使用持久化存储，如 SqliteSaver、PostgresSaver
checkpointer = MemorySaver()
```

### Checkpointer 类型对比

| 类型 | 存储位置 | 适用场景 | 持久化 |
|------|---------|---------|--------|
| `MemorySaver` | 内存 | 开发测试 | 否（进程结束即丢失） |
| `SqliteSaver` | SQLite 文件 | 单机生产 | 是 |
| `PostgresSaver` | PostgreSQL | 分布式生产 | 是 |

## 第三步：构建带记忆的聊天图

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.graph import MessagesState
from langgraph.checkpoint.memory import MemorySaver

# 创建模型
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

# 定义聊天节点
def chatbot(state: MessagesState) -> dict:
    """简单的聊天机器人节点"""
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

# 构建图
workflow = StateGraph(MessagesState)
workflow.add_node("chatbot", chatbot)
workflow.add_edge(START, "chatbot")
workflow.add_edge("chatbot", END)

# 创建检查点保存器
checkpointer = MemorySaver()

# 编译时传入 checkpointer
app = workflow.compile(checkpointer=checkpointer)
```

> **关键点**：编译时必须传入 `checkpointer` 参数，否则图不会保存状态，多轮对话无法工作。

## 第四步：多轮对话演示

调用时必须提供 `thread_id` 配置，用于区分不同的对话线程：

```python
from langchain_core.messages import HumanMessage

# 配置：指定对话线程 ID
# 同一个 thread_id 的调用会共享对话历史
config = {"configurable": {"thread_id": "user_xiaoming"}}

# ===== 第 1 轮对话 =====
user_input = "你好，我叫小明，我是一名 Python 开发者。"
result = app.invoke(
    {"messages": [HumanMessage(content=user_input)]},
    config=config
)
print(f"用户: {user_input}")
print(f"AI: {result['messages'][-1].content}")
# AI: 你好小明！很高兴认识你。作为一名 Python 开发者...

# ===== 第 2 轮对话（AI 应该记得用户的名字） =====
user_input = "你还记得我叫什么名字吗？"
result = app.invoke(
    {"messages": [HumanMessage(content=user_input)]},
    config=config
)
print(f"\n用户: {user_input}")
print(f"AI: {result['messages'][-1].content}")
# AI: 当然记得，你叫小明！你是一名 Python 开发者。

# ===== 第 3 轮对话 =====
user_input = "推荐一些适合我的学习资源"
result = app.invoke(
    {"messages": [HumanMessage(content=user_input)]},
    config=config
)
print(f"\n用户: {user_input}")
print(f"AI: {result['messages'][-1].content}")
# AI: 作为 Python 开发者，我推荐以下资源...（会结合上下文回答）
```

### 不同线程互不干扰

```python
# 使用不同的 thread_id → 完全独立的对话
config_other = {"configurable": {"thread_id": "user_xiaohong"}}

result = app.invoke(
    {"messages": [HumanMessage(content="我叫小红")]},
    config=config_other
)
# AI 不会知道小明的信息，两个线程完全隔离
```

## 完整可运行代码

```python
"""
多轮对话 Agent 完整实现
支持对话记忆和消息裁剪
"""
import os
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import MemorySaver

# ========== 1. 创建模型 ==========
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

# ========== 2. 定义聊天节点 ==========
def chatbot(state: MessagesState) -> dict:
    """聊天机器人节点"""
    # 添加系统提示词
    system_msg = SystemMessage(content="你是一个友好的 AI 助手，善于记住用户的信息。")
    messages = [system_msg] + state["messages"]
    response = llm.invoke(messages)
    return {"messages": [response]}

# ========== 3. 构建图 ==========
workflow = StateGraph(MessagesState)
workflow.add_node("chatbot", chatbot)
workflow.add_edge(START, "chatbot")
workflow.add_edge("chatbot", END)

# ========== 4. 添加记忆 ==========
checkpointer = MemorySaver()
app = workflow.compile(checkpointer=checkpointer)

# ========== 5. 交互式对话 ==========
def chat(user_input: str, thread_id: str = "default") -> str:
    """发送消息并获取回复"""
    config = {"configurable": {"thread_id": thread_id}}
    result = app.invoke(
        {"messages": [HumanMessage(content=user_input)]},
        config=config
    )
    return result["messages"][-1].content

if __name__ == "__main__":
    print("=== 多轮对话演示 ===\n")
    
    # 第 1 轮
    reply = chat("你好，我叫小明，我今年 25 岁。", "demo_thread")
    print(f"用户: 你好，我叫小明，我今年 25 岁。")
    print(f"AI: {reply}\n")
    
    # 第 2 轮
    reply = chat("我叫什么名字？多大了？", "demo_thread")
    print(f"用户: 我叫什么名字？多大了？")
    print(f"AI: {reply}\n")
    
    # 第 3 轮
    reply = chat("帮我写一个 Python 的 hello world", "demo_thread")
    print(f"用户: 帮我写一个 Python 的 hello world")
    print(f"AI: {reply}\n")
```

## 消息裁剪：处理长对话

随着对话轮次增多，消息列表会越来越长，最终超出模型的上下文窗口限制。需要定期裁剪历史消息。

### 方案一：保留最近 N 条消息

```python
from langchain_core.messages import RemoveMessage

def trim_messages_node(state: MessagesState) -> dict:
    """裁剪消息：只保留最近 10 条"""
    messages = state["messages"]
    max_messages = 10
    
    if len(messages) > max_messages:
        # 保留最近的消息
        recent = messages[-max_messages:]
        # 删除旧消息（通过 RemoveMessage 实现）
        removals = [
            RemoveMessage(id=msg.id) 
            for msg in messages[:-max_messages]
        ]
        return {"messages": removals}
    
    return {}
```

### 方案二：用 LLM 摘要替代旧消息

```python
def summarize_and_trim(state: MessagesState) -> dict:
    """用摘要替代旧消息，保留对话要点"""
    messages = state["messages"]
    max_messages = 10
    
    if len(messages) <= max_messages:
        return {}
    
    # 提取需要摘要的旧消息
    old_messages = messages[:-max_messages]
    recent_messages = messages[-max_messages:]
    
    # 让 LLM 生成摘要
    summary_prompt = (
        "请将以下对话内容总结为简洁的摘要，保留关键信息：\n\n"
        + "\n".join([f"{m.type}: {m.content}" for m in old_messages])
    )
    summary = llm.invoke(summary_prompt)
    
    # 删除旧消息，插入摘要
    from langchain_core.messages import SystemMessage, RemoveMessage
    removals = [RemoveMessage(id=msg.id) for msg in old_messages]
    summary_msg = SystemMessage(content=f"[历史对话摘要] {summary.content}")
    
    return {"messages": removals + [summary_msg]}
```

### 方案三：在图中集成裁剪节点

```python
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import MemorySaver
from langchain_core.messages import RemoveMessage

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

def chatbot(state: MessagesState) -> dict:
    """聊天节点"""
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

def trim_messages(state: MessagesState) -> dict:
    """裁剪节点：保留最近 6 条消息"""
    messages = state["messages"]
    if len(messages) > 6:
        removals = [RemoveMessage(id=m.id) for m in messages[:-6]]
        return {"messages": removals}
    return {}

# 构建带裁剪的图
workflow = StateGraph(MessagesState)
workflow.add_node("chatbot", chatbot)
workflow.add_node("trim", trim_messages)

# 流程：START → chatbot → trim → END
workflow.add_edge(START, "chatbot")
workflow.add_edge("chatbot", "trim")
workflow.add_edge("trim", END)

checkpointer = MemorySaver()
app = workflow.compile(checkpointer=checkpointer)
```

执行流程：

```
START → chatbot → trim → END
          │          │
          │          └─ 如果消息 > 6 条，删除最旧的
          └─ 生成回复
```

### 方案四：使用 langchain_core 的 trim_messages 工具函数

```python
from langchain_core.messages import trim_messages

def chatbot_with_trim(state: MessagesState) -> dict:
    """在调用模型前裁剪消息"""
    # 保留最近 10 条消息（保留系统消息）
    trimmed = trim_messages(
        state["messages"],
        max_tokens=10,          # 最多保留 10 条
        strategy="last",        # 保留最近的
        token_counter=len,      # 计数方式
        include_system=True,    # 始终保留系统消息
    )
    response = llm.invoke(trimmed)
    return {"messages": [response]}
```

## 查看对话历史

Checkpointer 保存了完整的状态，可以查看任意线程的对话历史：

```python
# 获取当前状态
config = {"configurable": {"thread_id": "demo_thread"}}
state = app.get_state(config)

print("=== 对话历史 ===")
for msg in state.values["messages"]:
    role = "用户" if msg.type == "human" else "AI"
    print(f"{role}: {msg.content}")
```

## 时间旅行

Checkpointer 还支持"时间旅行"——回到对话的任意历史节点：

```python
# 获取历史快照
history = list(app.get_state_history(config))

print(f"共有 {len(history)} 个检查点")

# 回到第 2 轮对话的状态
# （索引 0 是最新状态，索引越大越早）
old_state = history[2]  # 第 2 轮后的状态
print(f"那个时刻的消息数: {len(old_state.values['messages'])}")
```

> 详见 [[langchain/modules/langchain_checkpoints/时间旅行|时间旅行详解]]

## 生产环境配置

### SQLite 持久化

```python
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

# 创建 SQLite 连接
conn = sqlite3.connect("checkpoints.db")
checkpointer = SqliteSaver(conn)

app = workflow.compile(checkpointer=checkpointer)
# 现在即使重启程序，对话历史也不会丢失
```

### PostgreSQL 持久化

```python
from langgraph.checkpoint.postgres import PostgresSaver

# 使用 PostgreSQL 连接
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost:5432/langgraph"
)

app = workflow.compile(checkpointer=checkpointer)
```

## 相关页面

- [[langgraph/tutorial/ReAct_Agent实现|ReAct Agent 实现]] — 基础 Agent 构建
- [[langgraph/modules/langgraph_state/概述|状态管理概述]] — 状态系统详解
- [[langchain/modules/langchain_checkpoints/时间旅行|时间旅行]] — 历史状态回溯
- [[langchain/modules/langchain_checkpoints/多轮对话与thread_id|thread_id 详解]] — 线程 ID 管理
- [[langchain/modules/langchain_messages/RemoveMessage历史裁剪|RemoveMessage 详解]] — 消息删除机制
- [[langgraph/tutorial/RAG_Agent|RAG Agent]] — 结合检索增强生成
