---
tags: [langchain, module, runnables, history, 消息历史, 对话记忆, 已弃用]
type: concept
created: 2026-07-11
updated: 2026-07-11
deprecated: true
deprecated_reason: "已被 create_agent + checkpointer 方案取代"
deprecated_alternative: "langchain.agents.create_agent(checkpointer=...)"
---

# RunnableWithMessageHistory

> ⚠️ **已弃用**：`RunnableWithMessageHistory` 已被 `create_agent` + `checkpointer` 方案取代。官方文档不再推荐使用此方式管理对话历史。新项目应直接使用 `create_agent(checkpointer=InMemorySaver())` 或 `PostgresSaver()`。本页保留仅供理解旧代码。

`RunnableWithMessageHistory` 是 `langchain_core.runnables.history` 中的类，曾用于为 LCEL 链自动管理**对话消息历史**。它包装一个 Runnable，在每次调用时自动加载历史消息、执行链、再将新消息写入历史存储。

## 是什么

当你用 LCEL 构建一个带对话历史的链时，通常需要手动管理消息历史：加载 → 拼入 prompt → 调 LLM → 保存新消息。`RunnableWithMessageHistory` 把这个过程自动化：

```
调用 invoke()
    ↓
自动从存储加载历史消息
    ↓
将历史注入 prompt（通过 MessagesPlaceholder）
    ↓
执行链（prompt | model | parser）
    ↓
自动将本轮对话（用户输入 + AI 回答）保存到存储
```

## 导入

```python
from langchain_core.runnables.history import RunnableWithMessageHistory
```

## 构造函数参数

| 参数 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `runnable` | `Runnable` | ✅ | 被包装的链（通常包含 `MessagesPlaceholder`） |
| `get_session_history` | `Callable` | ✅ | 根据 session_id 返回历史存储对象的函数 |
| `input_messages_key` | `str` | ❌ | dict 输入时，用户消息所在的 key |
| `output_messages_key` | `str` | ❌ | dict 输出时，AI 回答所在的 key |
| `history_messages_key` | `str` | ❌ | dict 输入时，历史消息所在的 key |
| `history_factory_config` | `list[ConfigurableFieldSpec]` | ❌ | 自定义 session 标识配置 |

## 基础用法

### 最简单的链（列表输入）

```python
from langchain_core.chat_history import BaseChatMessageHistory, InMemoryHistory
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

# 1. 构建包含历史占位符的链
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}"),
])
chain = prompt | model

# 2. 定义历史存储（内存版，适合开发）
store = {}

def get_session_history(session_id: str) -> BaseChatMessageHistory:
    """根据 session_id 获取或创建历史存储"""
    if session_id not in store:
        store[session_id] = InMemoryHistory()
    return store[session_id]

# 3. 包装链
chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history=get_session_history,
    input_messages_key="input",
    history_messages_key="history",
)

# 4. 调用（必须传 config 中的 session_id）
response = chain_with_history.invoke(
    {"input": "我叫小明"},
    config={"configurable": {"session_id": "user-001"}},
)
# AI: "你好小明！有什么可以帮你的？"

# 第二轮 —— 自动带上之前的对话历史
response = chain_with_history.invoke(
    {"input": "我叫什么名字？"},
    config={"configurable": {"session_id": "user-001"}},
)
# AI: "你叫小明。"  ← 从历史中读取了第一轮对话
```

### 纯消息列表输入（不需要 dict）

如果链只接受消息列表（不需要其他变量），可以省略 key 参数：

```python
prompt = ChatPromptTemplate.from_messages([
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}"),
])
chain = prompt | model

chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history=get_session_history,
    # 不需要指定 key，因为输入就是消息列表
)

# 调用时传消息列表
response = chain_with_history.invoke(
    [{"role": "user", "content": "你好"}],
    config={"configurable": {"session_id": "session-1"}},
)
```

## get_session_history 详解

这个函数是历史管理的核心。它接收 `session_id`，返回一个 `BaseChatMessageHistory` 实例：

```python
from langchain_core.chat_history import BaseChatMessageHistory, InMemoryHistory

# === 方式1：内存存储（开发用）===
store = {}

def get_session_history(session_id: str) -> BaseChatMessageHistory:
    if session_id not in store:
        store[session_id] = InMemoryHistory()
    return store[session_id]

# === 方式2：PostgreSQL 持久化（生产用）===
from langchain_postgres import PostgresChatMessageHistory

def get_session_history(session_id: str) -> BaseChatMessageHistory:
    return PostgresChatMessageHistory(
        table_name="chat_history",
        session_id=session_id,
        connection_string="postgresql://user:pass@localhost:5432/db",
    )

# === 方式3：Redis 持久化 ===
from langchain_redis import RedisChatMessageHistory

def get_session_history(session_id: str) -> BaseChatMessageHistory:
    return RedisChatMessageHistory(
        session_id=session_id,
        url="redis://localhost:6379",
        ttl=3600,  # 1小时后过期
    )
```

| 存储后端 | 导入 | 适用场景 |
|---------|------|---------|
| `InMemoryHistory` | `langchain_core.chat_history` | 开发/测试，进程重启丢失 |
| `PostgresChatMessageHistory` | `langchain_postgres` | 生产环境，关系型数据库 |
| `RedisChatMessageHistory` | `langchain_redis` | 生产环境，带 TTL 的缓存 |
| `SQLChatMessageHistory` | `langchain_community` | SQLite / MySQL |
| `MongoDBChatMessageHistory` | `langchain_mongodb` | MongoDB 文档存储 |

## session_id 与 config

`session_id` 通过 `config["configurable"]` 传入，用于区分不同的对话会话：

```python
# 同一个 session_id → 同一段对话，历史会累积
chain_with_history.invoke(
    {"input": "第一轮问题"},
    config={"configurable": {"session_id": "conversation-A"}},
)
chain_with_history.invoke(
    {"input": "第二轮问题"},
    config={"configurable": {"session_id": "conversation-A"}},
)
# 第二轮能记住第一轮的内容

# 不同 session_id → 不同的对话
chain_with_history.invoke(
    {"input": "新问题"},
    config={"configurable": {"session_id": "conversation-B"}},
)
# conversation-B 不知道 conversation-A 的历史
```

### 自定义 session 标识（history_factory_config）

默认只需要 `session_id`。如果需要额外的标识维度（比如 user_id + session_id）：

```python
from langchain_core.runnables import ConfigurableFieldSpec

def get_history(user_id: str, session_id: str) -> BaseChatMessageHistory:
    key = f"{user_id}:{session_id}"
    if key not in store:
        store[key] = InMemoryHistory()
    return store[key]

chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history=get_history,
    input_messages_key="input",
    history_messages_key="history",
    history_factory_config=[
        ConfigurableFieldSpec(
            id="user_id",
            annotation=str,
            name="User ID",
            description="用户唯一标识",
            default="",
        ),
        ConfigurableFieldSpec(
            id="session_id",
            annotation=str,
            name="Session ID",
            description="对话会话标识",
            default="",
        ),
    ],
)

# 调用时传两个标识
chain_with_history.invoke(
    {"input": "你好"},
    config={"configurable": {"user_id": "u-123", "session_id": "s-456"}},
)
```

## 完整实战示例

### RAG 链 + 对话历史

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.chat_history import BaseChatMessageHistory, InMemoryHistory
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# 组件初始化
model = ChatOpenAI(model="gpt-4o-mini")
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(embedding_function=embeddings, collection_name="docs")
retriever = vectorstore.as_retriever()

# 带历史的 RAG prompt
prompt = ChatPromptTemplate.from_messages([
    ("system", "根据以下上下文回答问题。\n\n上下文：{context}"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{question}"),
])

# RAG 链
def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    RunnablePassthrough.assign(context=lambda x: retriever.invoke(x["question"]) | format_docs)
    | prompt
    | model
    | StrOutputParser()
)

# 包装历史
store = {}
def get_session_history(session_id: str) -> BaseChatMessageHistory:
    if session_id not in store:
        store[session_id] = InMemoryHistory()
    return store[session_id]

rag_with_history = RunnableWithMessageHistory(
    rag_chain,
    get_session_history=get_session_history,
    input_messages_key="question",
    history_messages_key="history",
)

# 多轮对话
config = {"configurable": {"session_id": "rag-session-1"}}

rag_with_history.invoke(
    {"question": "LangChain 的 RAG 是什么？"},
    config=config,
)

rag_with_history.invoke(
    {"question": "它和普通搜索有什么区别？"},  # 能理解"它"指的是 RAG
    config=config,
)
```

## RunnableWithMessageHistory vs create_agent + checkpointer

两者都能"记住对话"，但工作层级和能力差距很大。知识库 memory 部分展示的是 `create_agent + checkpointer` 方案，而 `RunnableWithMessageHistory` 是另一种管理对话历史的方式。

### 核心定位差异

**RunnableWithMessageHistory** 是一个"消息历史的自动存取器"——它包装一个 LCEL 链，帮你省掉手动加载/保存消息的代码。

**create_agent + checkpointer** 是一个"有状态的 Agent 框架"——管理的不只是消息，而是整个执行状态（消息 + 工具调用记录 + 自定义字段）。

### 代码对比

```python
# ===== RunnableWithMessageHistory — 简单的链 + 消息历史 =====
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.chat_history import InMemoryHistory
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages([
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}"),
])
chain = prompt | model

chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history=lambda sid: InMemoryHistory(),
    input_messages_key="input",
    history_messages_key="history",
)

# 只存消息，不管其他
chain_with_history.invoke(
    {"input": "你好"},
    config={"configurable": {"session_id": "s1"}},
)
```

```python
# ===== create_agent + checkpointer — Agent + 完整状态管理 =====
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model=llm,
    tools=[get_weather, search],       # 有工具
    checkpointer=InMemorySaver(),
)

# 存的是完整状态快照：消息 + 工具调用 + 自定义字段
agent.invoke(
    {"messages": [HumanMessage(content="东京天气？")]},
    config={"configurable": {"thread_id": "t1"}},
)
```

### 详细对比

| 维度 | RunnableWithMessageHistory | create_agent + checkpointer |
|------|--------------------------|---------------------------|
| **所属模块** | `langchain_core.runnables.history` | `langchain.agents` + `langgraph.checkpoint` |
| **适用对象** | 普通 LCEL 链（线性流水线） | Agent（带工具调用的循环） |
| **存储内容** | 只存消息（HumanMessage + AIMessage） | 存完整状态快照（消息 + 工具调用记录 + 自定义字段） |
| **存储接口** | `BaseChatMessageHistory`（InMemoryHistory 等） | `BaseCheckpointSaver`（InMemorySaver、PostgresSaver 等） |
| **会话标识** | `session_id` | `thread_id` |
| **工具调用** | 不支持（链没有工具循环） | 支持（Agent 自动调度工具） |
| **自定义状态** | 不支持 | 支持（扩展 AgentState 加自定义字段） |
| **时间旅行** | 不支持 | 支持（回溯到任意 checkpoint） |
| **消息裁剪** | 需手动加 trim_messages | 通过 middleware 自动管理 |
| **配置复杂度** | 低 | 较高 |

### 存储接口差异

```python
# RunnableWithMessageHistory 使用的存储接口
from langchain_core.chat_history import BaseChatMessageHistory, InMemoryHistory

# 存储的是纯消息列表
history = InMemoryHistory()
history.add_message(HumanMessage(content="你好"))
history.add_message(AIMessage(content="你好！"))
history.messages  # [HumanMessage(...), AIMessage(...)]
```

```python
# create_agent 使用的存储接口
from langgraph.checkpoint.memory import InMemorySaver

# 存储的是完整状态快照（包含 messages + 自定义字段）
checkpointer = InMemorySaver()
# 内部存储结构：
# {
#   "messages": [HumanMessage, AIMessage, ToolMessage, ...],
#   "user_id": "u123",          ← 自定义字段
#   "request_count": 5,         ← 自定义字段
#   ...
# }
```

### 选型决策

```
需要对话历史？
├── 不需要工具调用（纯问答、RAG、翻译）
│   └── → RunnableWithMessageHistory（轻量、简单）
│
├── 需要工具调用（搜索、计算、数据库查询）
│   └── → create_agent + checkpointer
│
└── 需要时间旅行 / 复杂状态管理 / 人机交互中断
    └── → create_agent + PostgresSaver
```

**实际建议**：现在大多数场景都用 `create_agent`，因为即使是简单的对话也可能需要工具（搜索、计算器等）。`RunnableWithMessageHistory` 更多出现在不需要工具的轻量场景中，或者在 Agent 框架流行之前的老代码里。

## 常见陷阱

### 陷阱 1：忘记传 session_id

```python
# ❌ 报错：config 中缺少 session_id
chain_with_history.invoke({"input": "你好"})
# KeyError 或 ValueError

# ✅ 正确：必须在 config 中传 session_id
chain_with_history.invoke(
    {"input": "你好"},
    config={"configurable": {"session_id": "my-session"}},
)
```

### 陷阱 2：prompt 中没有 MessagesPlaceholder

```python
# ❌ 历史消息无处注入
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是助手"),
    ("human", "{input}"),
    # 缺少 MessagesPlaceholder！
])
chain = prompt | model
# RunnableWithMessageHistory 会报错或忽略历史

# ✅ 正确：必须包含 MessagesPlaceholder
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是助手"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}"),
])
```

### 陷阱 3：key 名不匹配

```python
# ❌ prompt 中的占位符名和 key 参数不一致
prompt = ChatPromptTemplate.from_messages([
    MessagesPlaceholder(variable_name="chat_history"),  # 这里是 chat_history
    ("human", "{question}"),
])

chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history=get_session_history,
    input_messages_key="question",
    history_messages_key="history",  # ❌ 应该是 "chat_history"
)

# ✅ 正确：保持一致
chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history=get_session_history,
    input_messages_key="question",
    history_messages_key="chat_history",  # ✅ 匹配
)
```

### 陷阱 4：历史无限增长

随着对话轮数增加，历史消息会越来越多，可能导致超出模型的上下文窗口。解决方案：

```python
from langchain_core.messages import trim_messages

# 在链中加入消息裁剪
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是助手"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}"),
])

# 方法1：在 get_session_history 返回前裁剪
def get_session_history(session_id: str) -> BaseChatMessageHistory:
    if session_id not in store:
        store[session_id] = InMemoryHistory()
    return store[session_id][[

# 方法2：在链中用 RunnableLambda 裁剪
from langchain_core.runnables import RunnableLambda

trimmer = RunnableLambda(
    lambda x: trim_messages(x, max_tokens=2000, strategy="last")
)
chain = prompt | trimmer | model  # 在 prompt 之后裁剪
```

## 相关页面

- [[langchain/modules/langchain_runnables/概述|← 返回 runnables 模块]]
- [[langchain/modules/langchain_prompts/MessagesPlaceholder|MessagesPlaceholder - 消息占位符]]
- [[langchain/modules/langchain_checkpoints/概述|checkpoints 模块 - Agent 状态持久化]]
- [[langchain/modules/langchain_checkpoints/多轮对话与thread_id|多轮对话与 thread_id]]
- [[langchain/memory/短期记忆|短期记忆概念]]
