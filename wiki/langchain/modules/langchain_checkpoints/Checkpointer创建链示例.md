---
tags: [langchain, checkpoints, example, checkpointer, chain, 示例代码]
type: example
created: 2026-07-11
updated: 2026-07-11
---

# Checkpointer 创建链完整示例

> 使用 `create_agent` + `checkpointer` 创建带对话历史的链。这是当前官方推荐的方式，取代了已弃用的 `RunnableWithMessageHistory`。

## 最简示例：内存 Checkpointer

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.messages import HumanMessage

# 1. 创建 Agent，注入 checkpointer
agent = create_agent(
    model="openai:gpt-4o-mini",
    checkpointer=InMemorySaver(),
)

# 2. 定义 thread_id（对话标识）
config = {"configurable": {"thread_id": "thread-001"}}

# 3. 第一轮对话
result = agent.invoke(
    {"messages": [HumanMessage(content="你好，我叫小明")]},
    config=config,
)
print(result["messages"][-1].content)
# "你好小明！很高兴认识你。"

# 4. 第二轮对话 —— 自动记住上一轮的内容
result = agent.invoke(
    {"messages": [HumanMessage(content="我叫什么名字？")]},
    config=config,
)
print(result["messages"][-1].content)
# "你叫小明。"  ← 从 checkpointer 中恢复了历史
```

## 带系统提示词的链

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.messages import HumanMessage

agent = create_agent(
    model="openai:gpt-4o-mini",
    system_prompt="你是一个专业的 Python 编程导师，用简洁的中文回答，多给代码示例。",
    checkpointer=InMemorySaver(),
)

config = {"configurable": {"thread_id": "python-tutor-001"}}

result = agent.invoke(
    {"messages": [HumanMessage(content="什么是装饰器？")]},
    config=config,
)
print(result["messages"][-1].content)

# 追问 —— 保持上下文
result = agent.invoke(
    {"messages": [HumanMessage(content="能给一个带参数的装饰器例子吗？")]},
    config=config,
)
print(result["messages"][-1].content)
```

## 带工具的链

```python
from langchain.agents import create_agent
from langchain.tools import tool
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.messages import HumanMessage

# 定义工具
@tool
def calculate(expression: str) -> str:
    """计算数学表达式，如 '2 + 3 * 4'"""
    try:
        return str(eval(expression))
    except Exception as e:
        return f"计算错误: {e}"

@tool
def get_current_time() -> str:
    """获取当前时间"""
    from datetime import datetime
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

# 创建带工具的 Agent
agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[calculate, get_current_time],
    system_prompt="你是一个有用的助手，需要时可以调用工具。",
    checkpointer=InMemorySaver(),
)

config = {"configurable": {"thread_id": "tool-agent-001"}}

# Agent 会自动决定何时调用工具
result = agent.invoke(
    {"messages": [HumanMessage(content="计算 (123 + 456) * 789 的结果")]},
    config=config,
)
print(result["messages"][-1].content)
# "计算结果是 457299。"

# 追问 —— 记住之前的对话
result = agent.invoke(
    {"messages": [HumanMessage(content="刚才那个结果除以 3 是多少？")]},
    config=config,
)
print(result["messages"][-1].content)
# "457299 除以 3 等于 152433。"
```

## 多用户隔离：不同 thread_id

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.messages import HumanMessage

agent = create_agent(
    model="openai:gpt-4o-mini",
    checkpointer=InMemorySaver(),
)

# 用户 A 的对话
config_a = {"configurable": {"thread_id": "user-alice"}}
agent.invoke(
    {"messages": [HumanMessage(content="我是 Alice，我喜欢蓝色")]},
    config=config_a,
)

# 用户 B 的对话
config_b = {"configurable": {"thread_id": "user-bob"}}
agent.invoke(
    {"messages": [HumanMessage(content="我是 Bob，我喜欢红色")]},
    config=config_b,
)

# 用户 A 追问 —— 只记得 Alice 的信息
result = agent.invoke(
    {"messages": [HumanMessage(content="我叫什么？我喜欢什么颜色？")]},
    config=config_a,
)
print(result["messages"][-1].content)
# "你叫 Alice，你喜欢蓝色。"

# 用户 B 追问 —— 只记得 Bob 的信息
result = agent.invoke(
    {"messages": [HumanMessage(content="我叫什么？我喜欢什么颜色？")]},
    config=config_b,
)
print(result["messages"][-1].content)
# "你叫 Bob，你喜欢红色。"
```

## 生产环境：PostgreSQL Checkpointer

```python
from langchain.agents import create_agent
from langgraph.checkpoint.postgres import PostgresSaver
from langchain_core.messages import HumanMessage

# PostgreSQL 连接字符串
DB_URI = "postgresql://user:password@localhost:5432/mydb"

# 创建 checkpointer 并初始化表结构
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()  # 自动创建 checkpoints 表

    agent = create_agent(
        model="openai:gpt-4o-mini",
        checkpointer=checkpointer,
    )

    config = {"configurable": {"thread_id": "production-001"}}

    result = agent.invoke(
        {"messages": [HumanMessage(content="你好，记住我喜欢 Python")]},
        config=config,
    )
    print(result["messages"][-1].content)

# checkpointer 关闭后，数据已持久化到 PostgreSQL
# 下次启动程序时，用相同的 thread_id 可以恢复对话
```

## 生产环境：SQLite Checkpointer（轻量持久化）

```python
import sqlite3
from langchain.agents import create_agent
from langgraph.checkpoint.sqlite import SqliteSaver
from langchain_core.messages import HumanMessage

# SQLite 连接
connection = sqlite3.connect("chat_history.db", check_same_thread=False)
checkpointer = SqliteSaver(connection)
checkpointer.setup()

agent = create_agent(
    model="openai:gpt-4o-mini",
    checkpointer=checkpointer,
)

config = {"configurable": {"thread_id": "sqlite-001"}}

result = agent.invoke(
    {"messages": [HumanMessage(content="记住我喜欢喝咖啡")]},
    config=config,
)
print(result["messages"][-1].content)

# 数据保存在 chat_history.db 文件中，重启程序后仍然可用
```

## 自定义状态：扩展 AgentState

```python
from langchain.agents import create_agent, AgentState
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.messages import HumanMessage

# 扩展状态，添加自定义字段
class MyState(AgentState):
    user_name: str
    request_count: int

agent = create_agent(
    model="openai:gpt-4o-mini",
    state_schema=MyState,
    checkpointer=InMemorySaver(),
)

config = {"configurable": {"thread_id": "custom-state-001"}}

result = agent.invoke(
    {
        "messages": [HumanMessage(content="你好")],
        "user_name": "小明",
        "request_count": 1,
    },
    config=config,
)

# 自定义字段也会被 checkpointer 持久化
print(result["messages"][-1].content)
```

## 消息裁剪：防止历史过长

```python
from langchain.agents import create_agent
from langchain.agents.middleware import before_model
from langchain.messages import RemoveMessage, REMOVE_ALL_MESSAGES
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.messages import HumanMessage

# 定义裁剪中间件：只保留最近 10 条消息
@before_model
def trim_history(state, runtime):
    messages = state["messages"]
    if len(messages) <= 10:
        return None
    # 保留最近 10 条
    recent = messages[-10:]
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)] + recent}

agent = create_agent(
    model="openai:gpt-4o-mini",
    checkpointer=InMemorySaver(),
    middleware=[trim_history],
)

config = {"configurable": {"thread_id": "trim-001"}}

# 即使对话很多轮，历史也最多保留 10 条消息
for i in range(20):
    agent.invoke(
        {"messages": [HumanMessage(content=f"第 {i+1} 轮对话")]},
        config=config,
    )
```

## 完整实战：带工具 + 裁剪 + 持久化的对话机器人

```python
import sqlite3
from langchain.agents import create_agent
from langchain.agents.middleware import before_model
from langchain.tools import tool
from langchain.messages import RemoveMessage, REMOVE_ALL_MESSAGES
from langgraph.checkpoint.sqlite import SqliteSaver
from langchain_core.messages import HumanMessage

# === 工具定义 ===
@tool
def search_knowledge_base(query: str) -> str:
    """搜索知识库获取相关信息"""
    # 实际项目中这里会连接向量数据库
    return f"关于 '{query}' 的信息：LangChain 是一个 LLM 应用开发框架..."

@tool
def get_user_preference(user_id: str) -> str:
    """获取用户偏好设置"""
    return f"用户 {user_id} 偏好：中文回答，简洁风格"

# === 中间件：裁剪历史 ===
@before_model
def trim_history(state, runtime):
    messages = state["messages"]
    if len(messages) > 20:
        recent = messages[-20:]
        return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)] + recent}
    return None

# === 初始化 ===
connection = sqlite3.connect("bot.db", check_same_thread=False)
checkpointer = SqliteSaver(connection)
checkpointer.setup()

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search_knowledge_base, get_user_preference],
    system_prompt="你是一个智能助手，用中文回答。需要时可以搜索知识库。",
    checkpointer=checkpointer,
    middleware=[trim_history],
)

# === 对话循环 ===
def chat(user_input: str, thread_id: str):
    config = {"configurable": {"thread_id": thread_id}}
    result = agent.invoke(
        {"messages": [HumanMessage(content=user_input)]},
        config=config,
    )
    return result["messages"][-1][[.content

# 使用
print(chat("LangChain 是什么？", "user-001"))
# → Agent 可能调用 search_knowledge_base，返回知识库内容

print(chat("用一句话总结", "user-001"))
# → 记住上一轮的上下文，直接总结

print(chat("你好", "user-002"))
# → 不同的 thread_id，完全隔离的对话
```

## 相关页面

- [[langchain/modules/langchain_checkpoints/概述|checkpoints 模块概述]]
- [[langchain/modules/langchain_checkpoints/多轮对话与thread_id|多轮对话与 thread_id]]
- [[langchain/modules/langchain_checkpoints/时间旅行|时间旅行]]
- [[langchain/modules/langchain_checkpoints/人机交互中断与恢复|人机交互中断与恢复]]
- [[langchain/memory/短期记忆|短期记忆概念]]
- [[langchain/modules/langchain_runnables/RunnableWithMessageHistory|RunnableWithMessageHistory（已弃用）]]
