---
tags: [langgraph, checkpoint, thread_id, 会话管理, 多用户, 配置]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# thread_id 与会话管理

## 概述

`thread_id` 是 LangGraph 中用于隔离不同对话/会话的核心机制。通过为不同的对话分配不同的 `thread_id`，可以实现：

- **多轮对话**：同一个 `thread_id` 下的多次调用共享完整的历史状态
- **会话隔离**：不同 `thread_id` 之间的状态完全独立，互不干扰
- **多用户支持**：每个用户/会话使用独立的 `thread_id`

```
thread_id = "user-alice"
├── 调用1: "你好" → 状态1
├── 调用2: "我叫Alice" → 状态2（包含调用1的历史）
└── 调用3: "我叫什么？" → 状态3（包含调用1、2的历史）→ "你叫Alice"

thread_id = "user-bob"
├── 调用1: "你好" → 状态1（独立于 Alice）
└── 调用2: "我叫什么？" → 状态2 → "我不知道你的名字"（Bob 还没说过）
```

---

## Config 结构

`thread_id` 通过 `config` 参数传递给图的调用方法：

```python
# config 的标准结构
config = {
    "configurable": {
        "thread_id": "your-thread-id-here"
    }
}

# 在调用时传入
result = app.invoke(input_data, config=config)

# 也适用于 stream
for chunk in app.stream(input_data, config=config):
    print(chunk)

# 也适用于 ainvoke（异步）
result = await app.ainvoke(input_data, config=config)
```

### Config 的完整结构

```python
config = {
    "configurable": {
        "thread_id": "conversation-123",     # 会话ID（必需）
        "checkpoint_ns": "",                  # 检查点命名空间（可选，子图用）
        # 其他自定义配置也可以放在这里
    }
}
```

---

## 基本用法

### 单会话多轮对话

```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.checkpoint.memory import MemorySaver

def chatbot(state: MessagesState) -> dict:
    """简单回显机器人"""
    messages = state["messages"]
    last_msg = messages[-1].content if messages else ""
    return {"messages": [("assistant", f"你说了：{last_msg}，历史共{len(messages)}条消息")]}

# 构建图
graph = StateGraph(MessagesState)
graph.add_node("chatbot", chatbot)
graph.add_edge(START, "chatbot")
graph.add_edge("chatbot", END)

checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)

# 同一个 thread_id → 共享历史
config = {"configurable": {"thread_id": "my-conversation"}}

# 第1轮
r1 = app.invoke({"messages": [("user", "第一条消息")]}, config=config)
print(f"消息数: {len(r1['messages'])}")  # → 2（1条输入 + 1条回复）

# 第2轮
r2 = app.invoke({"messages": [("user", "第二条消息")]}, config=config)
print(f"消息数: {len(r2['messages'])}")  # → 4（之前的2条 + 1条输入 + 1条回复）

# 第3轮
r3 = app.invoke({"messages": [("user", "第三条消息")]}, config=config)
print(f"消息数: {len(r3['messages'])}")  # → 6
```

### 多会话隔离

```python
# 不同的 thread_id → 完全隔离
config_alice = {"configurable": {"thread_id": "alice-session"}}
config_bob = {"configurable": {"thread_id": "bob-session"}}

# Alice 的对话
app.invoke({"messages": [("user", "我是Alice")]}, config=config_alice)

# Bob 的对话
app.invoke({"messages": [("user", "我是Bob")]}, config=config_bob)

# Alice 看不到 Bob 的消息
alice_state = app.get_state(config_alice)
for msg in alice_state.values["messages"]:
    print(msg.content)
    # → "我是Alice"
    # → "你好Alice！..."
    # 没有 Bob 的消息！
```

---

## thread_id 设计策略

### 策略一：用户级隔离

```python
# 每个用户一个 thread_id（所有对话共享历史）
config = {"configurable": {"thread_id": f"user-{user_id}"}}
```

**适用**：每个用户只有一个对话上下文的场景

### 策略二：会话级隔离（推荐）

```python
# 每个会话一个 thread_id（同一用户可有多个独立会话）
config = {"configurable": {"thread_id": f"user-{user_id}-session-{session_id}"}}
```

**适用**：用户可能同时有多个独立对话的场景（如 ChatGPT 的多会话）

### 策略三：UUID 随机生成

```python
import uuid

# 每次新对话生成唯一 ID
thread_id = str(uuid.uuid4())
config = {"configurable": {"thread_id": thread_id}}
```

**适用**：不需要从 ID 推断用户信息，或需要全局唯一标识的场景

### 策略四：业务语义命名

```python
# 使用有业务含义的 ID
config = {
    "configurable": {
        "thread_id": f"support-ticket-{ticket_id}"
    }
}
```

**适用**：便于调试和追踪，ID 本身携带业务信息

---

## 多用户场景

### Web 应用中的会话管理

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.checkpoint.memory import MemorySaver

app = FastAPI()

# 编译图（全局共享一个 checkpointer）
checkpointer = MemorySaver()
graph_app = build_graph().compile(checkpointer=checkpointer)

class ChatRequest(BaseModel):
    user_id: str
    session_id: str
    message: str

@app.post("/chat")
async def chat(req: ChatRequest):
    # 为每个用户+会话组合生成唯一的 thread_id
    thread_id = f"{req.user_id}:{req.session_id}"
    config = {"configurable": {"thread_id": thread_id}}

    # 调用图
    result = graph_app.invoke(
        {"messages": [("user", req.message)]},
        config=config
    )

    # 返回最新的回复
    return {
        "reply": result["messages"][-1].content,
        "thread_id": thread_id,
        "message_count": len(result["messages"]),
    }

@app.get("/history/{user_id}/{session_id}")
async def get_history(user_id: str, session_id: str):
    """获取对话历史"""
    thread_id = f"{user_id}:{session_id}"
    config = {"configurable": {"thread_id": thread_id}}

    state = graph_app.get_state(config)
    if state is None:
        raise HTTPException(status_code=404, detail="会话不存在")

    return {
        "messages": [
            {"role": msg.type, "content": msg.content}
            for msg in state.values["messages"]
        ]
    }
```

### 多用户隔离验证

```python
import asyncio
from langgraph.checkpoint.memory import MemorySaver

async def simulate_users():
    """模拟多用户并发对话"""
    checkpointer = MemorySaver()
    app = build_graph().compile(checkpointer=checkpointer)

    async def user_chat(user_id: str, messages: list[str]):
        """模拟一个用户的多次对话"""
        config = {"configurable": {"thread_id": user_id}}
        results = []
        for msg in messages:
            result = await app.ainvoke(
                {"messages": [("user", msg)]},
                config=config
            )
            results.append(result["messages"][-1].content)
        return results

    # 并发模拟多个用户
    results = await asyncio.gather(
        user_chat("user-alice", ["你好", "我叫Alice"]),
        user_chat("user-bob", ["你好", "我叫Bob"]),
        user_chat("user-charlie", ["你好", "我叫Charlie"]),
    )

    # 验证隔离性
    alice_state = app.get_state({"configurable": {"thread_id": "user-alice"}})
    alice_messages = [m.content for m in alice_state.values["messages"]]
    assert "我叫Bob" not in str(alice_messages)  # Alice 看不到 Bob 的消息
    assert "我叫Charlie" not in str(alice_messages)
```

---

## 会话状态管理

### 获取会话状态

```python
# 获取当前状态
state = app.get_state(config)
print(f"当前值: {state.values}")
print(f"下一步: {state.next}")
print(f"配置: {state.config}")
```

### 获取状态历史

```python
# 遍历所有历史检查点
for snapshot in app.get_state_history(config):
    step = snapshot.metadata.get("step", "N/A")
    print(f"步骤 {step}:")
    print(f"  消息数: {len(snapshot.values.get('messages', []))}")
    print(f"  下一个节点: {snapshot.next}")
    print("---")
```

### 更新会话状态

```python
# 手动修改会话状态（用于人工介入、纠正等）
app.update_state(
    config,
    {"messages": [("user", "修正后的消息")]}
)

# 修改后继续执行
result = app.invoke(None, config=config)  # 传入 None 表示使用修改后的状态
```

### 删除/清空会话

```python
# LangGraph 没有直接的"删除会话"API
# 但可以通过以下方式实现：

# 方式一：使用新的 thread_id（旧的留在存储中，不再访问）
new_config = {"configurable": {"thread_id": "new-thread-id"}}

# 方式二：手动清理（取决于 Checkpointer 实现）
# 对于 PostgresSaver：
# cursor.execute("DELETE FROM checkpoints WHERE thread_id = %s", (thread_id,))

# 方式三：在节点中检查并重置
def check_and_reset(state: GraphState) -> dict:
    if state.get("should_reset"):
        return {
            "messages": [],  # 清空消息
            "should_reset": False,
        }
    return {}
```

---

## 高级用法

### 在 Config 中传递额外参数

```python
# 除了 thread_id，还可以在 configurable 中传递其他参数
config = {
    "configurable": {
        "thread_id": "my-session",
        "user_name": "Alice",        # 自定义参数
        "preferred_language": "zh",   # 自定义参数
    }
}

# 在节点中通过 state 或 config 访问
def my_node(state: GraphState, config: dict) -> dict:
    user_name = config["configurable"].get("user_name", "未知")
    lang = config["configurable"].get("preferred_language", "en")
    # 使用这些参数...
    return {"response": f"Hello {user_name} (lang: {lang})"}
```

### 动态 thread_id

```python
def get_or_create_thread(user_id: str, topic: str) -> str:
    """根据用户和主题获取或创建 thread_id"""
    # 可以用数据库查找已有的 thread
    # 如果没有，创建新的
    import uuid
    return f"{user_id}-{topic}-{uuid.uuid4().hex[:8]}"

# 使用
thread_id = get_or_create_thread("user-123", "tech-support")
config = {"configurable": {"thread_id": thread_id}}
```

### 会话元数据

```python
# 在状态中存储会话元数据
class ChatState(TypedDict):
    messages: Annotated[list, add_messages]
    session_started_at: str      # 会话开始时间
    last_active_at: str          # 最后活跃时间
    message_count: int           # 消息计数
    user_preferences: dict       # 用户偏好

def track_metadata(state: ChatState) -> dict:
    """跟踪会话元数据"""
    from datetime import datetime
    return {
        "last_active_at": datetime.now().isoformat(),
        "message_count": len(state["messages"]),
    }
```

---

## 常见问题

### Q: thread_id 有长度限制吗？

```
取决于 Checkpointer 实现：
- InMemorySaver：无限制（字典键）
- PostgresSaver：TEXT 类型，理论上无限制（建议 < 255 字符）
- SqliteSaver：TEXT 类型，理论上无限制（建议 < 255 字符）

建议：保持 thread_id 简短且有语义
```

### Q: 同一个 thread_id 可以有多个并发调用吗？

```
- InMemorySaver：可以，但可能产生竞态条件
- PostgresSaver：可以，通过事务保证一致性
- SqliteSaver：可以，但写入会串行化

建议：同一个 thread_id 避免真正的并发写入
```

### Q: thread_id 为空会怎样？

```python
# 不提供 thread_id 时，持久化不生效（等价于没有 checkpointer）
result = app.invoke({"messages": [("user", "你好")]})
# 每次调用都是独立的，没有历史

# 提供空字符串作为 thread_id
config = {"configurable": {"thread_id": ""}}
# 所有使用空 thread_id 的调用共享同一个历史（通常不是期望的行为）
```

---

## 最佳实践

1. **使用有意义的 thread_id**：包含用户ID和会话标识，便于调试
2. **会话级隔离**：推荐 `user_id:session_id` 格式
3. **不要共享 thread_id**：不同用户/会话必须使用不同的 thread_id
4. **定期清理**：对于长期运行的应用，清理过期的检查点数据
5. **使用 UUID**：当不需要从 ID 推断信息时，使用 UUID 保证唯一性
6. **错误处理**：处理 thread_id 不存在或无效的情况

```python
# 推荐的 thread_id 生成函数
import uuid
from datetime import datetime

def create_thread_id(user_id: str, session_type: str = "chat") -> str:
    """
    生成规范的 thread_id

    格式: {user_id}:{session_type}:{timestamp}:{short_uuid}
    示例: user-123:chat:20260711:a1b2c3d4
    """
    timestamp = datetime.now().strftime("%Y%m%d")
    short_id = uuid.uuid4().hex[:8]
    return f"{user_id}:{session_type}:{timestamp}:{short_id}"
```

---

## 相关页面

- [[langgraph/modules/langgraph_checkpoint/概述|持久化概述]]
- [[langgraph/modules/langgraph_checkpoint/InMemorySaver|InMemorySaver]]
- [[langgraph/modules/langgraph_checkpoint/PostgresSaver|PostgresSaver]]
- [[langgraph/modules/langgraph_checkpoint/SqliteSaver|SqliteSaver]]
