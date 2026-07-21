---
tags: [langgraph, checkpoint, SqliteSaver, SQLite, 轻量持久化, 文件存储]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# SqliteSaver（SQLite 检查点器）

## 概述

`SqliteSaver` 是基于 SQLite 的 Checkpointer 实现，提供轻量级的文件持久化。它将所有检查点存储在一个本地 SQLite 数据库文件中，无需外部数据库服务。

### 特点

- **文件存储**：数据保存在单个 SQLite 文件中
- **零服务**：不需要运行独立的数据库服务
- **持久化**：进程重启后数据保留
- **便携**：数据库就是一个文件，易于备份和迁移

### 适用场景

- 单机生产环境
- 桌面应用
- 嵌入式场景
- 需要持久化但不想部署 PostgreSQL 的场景
- 原型开发（比 InMemorySaver 多一层持久化）

### 不适用场景

- 高并发多进程写入（SQLite 写锁限制）
- 分布式部署（多机器无法共享文件）
- 大规模数据（SQLite 性能上限）

---

## 安装

```bash
# 安装 langgraph-checkpoint-sqlite 包
pip install langgraph-checkpoint-sqlite
```

---

## 导入与创建

```python
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

# 方式一：使用数据库文件路径
conn = sqlite3.connect("langgraph_checkpoints.db")
checkpointer = SqliteSaver(conn)

# 方式二：使用内存数据库（等价于 InMemorySaver，但接口一致）
conn = sqlite3.connect(":memory:")
checkpointer = SqliteSaver(conn)
```

---

## 基本用法

### 完整示例

```python
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.graph import StateGraph, MessagesState, START, END

# 定义节点
def chatbot(state: MessagesState) -> dict:
    last_msg = state["messages"][-1].content
    return {"messages": [("assistant", f"你说的是：{last_msg}")]}

# 构建图
graph = StateGraph(MessagesState)
graph.add_node("chatbot", chatbot)
graph.add_edge(START, "chatbot")
graph.add_edge("chatbot", END)

# 创建 SQLite Checkpointer
conn = sqlite3.connect("my_chatbot.db")
checkpointer = SqliteSaver(conn)

# 创建表（首次使用需要调用）
checkpointer.setup()

# 编译图
app = graph.compile(checkpointer=checkpointer)

# 使用
config = {"configurable": {"thread_id": "user-001"}}

# 第一轮对话
result = app.invoke(
    {"messages": [("user", "你好")]},
    config=config
)
print(result["messages"])

# 第二轮对话（历史保留）
result = app.invoke(
    {"messages": [("user", "上一轮我说了什么？")]},
    config=config
)
# LLM 能看到第一轮的消息

# 关闭连接
conn.close()
```

### 使用上下文管理器

```python
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver

# 推荐使用上下文管理器确保连接正确关闭
with sqlite3.connect("my_chatbot.db") as conn:
    checkpointer = SqliteSaver(conn)
    checkpointer.setup()
    app = graph.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "session-1"}}
    result = app.invoke(input_data, config=config)
    print(result)
# 退出 with 块后连接自动关闭
```

---

## setup() 方法

`setup()` 创建 SQLite 中所需的表结构：

```python
checkpointer.setup()
```

### 创建的表

```sql
-- 检查点表
CREATE TABLE IF NOT EXISTS checkpoints (
    thread_id TEXT NOT NULL,
    checkpoint_id TEXT NOT NULL,
    checkpoint_ns TEXT DEFAULT '',
    parent_checkpoint_id TEXT,
    type TEXT,
    checkpoint_json TEXT NOT NULL,
    metadata_json TEXT DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (thread_id, checkpoint_id, checkpoint_ns)
);

-- 写入记录表
CREATE TABLE IF NOT EXISTS checkpoint_writes (
    thread_id TEXT NOT NULL,
    checkpoint_id TEXT NOT NULL,
    checkpoint_ns TEXT DEFAULT '',
    task_id TEXT NOT NULL,
    idx INTEGER NOT NULL,
    channel TEXT NOT NULL,
    type TEXT,
    blob BLOB NOT NULL,
    PRIMARY KEY (thread_id, checkpoint_id, checkpoint_ns, task_id, idx)
);
```

> **注意**：SQLite 版本使用 TEXT 存储 JSON（而非 PostgreSQL 的 JSONB），因此查询性能略低。

---

## 数据库文件管理

### 文件位置

```python
# 相对路径（在当前工作目录下创建文件）
conn = sqlite3.connect("langgraph.db")

# 绝对路径（推荐，避免路径歧义）
import os
db_path = os.path.join(os.path.dirname(__file__), "data", "langgraph.db")
os.makedirs(os.path.dirname(db_path), exist_ok=True)
conn = sqlite3.connect(db_path)

# 用户目录
db_path = os.path.expanduser("~/my_app/langgraph.db")
os.makedirs(os.path.dirname(db_path), exist_ok=True)
conn = sqlite3.connect(db_path)
```

### 备份

```python
# SQLite 的优势：数据库就是一个文件，备份非常简单
import shutil

# 简单复制文件即可备份
shutil.copy2("langgraph.db", "langgraph_backup.db")

# 或使用 SQLite 的在线备份 API
import sqlite3

source = sqlite3.connect("langgraph.db")
backup = sqlite3.connect("langgraph_backup.db")
source.backup(backup)  # 在线备份，不锁库
backup.close()
source.close()
```

### 清理旧数据

```python
import sqlite3

conn = sqlite3.connect("langgraph.db")
cursor = conn.cursor()

# 删除30天前的检查点
cursor.execute("""
    DELETE FROM checkpoints
    WHERE created_at < datetime('now', '-30 days')
""")
deleted_count = cursor.rowcount
conn.commit()
print(f"已清理 {deleted_count} 条旧检查点")

conn.close()
```

---

## 并发限制

SQLite 的并发写入能力有限，需要注意：

### 读写并发

| 操作 | SQLite 行为 |
|------|-------------|
| 多读 | ✅ 无限制，可以并发读取 |
| 单写 | ⚠️ 同一时刻只允许一个写入 |
| 读写混合 | ⚠️ 写入时可能阻塞读取 |

### 应对策略

```python
import sqlite3

# 1. 设置超时时间（默认5秒）
conn = sqlite3.connect("langgraph.db", timeout=30)

# 2. 使用 WAL 模式（提升读写并发性能）
conn.execute("PRAGMA journal_mode=WAL")
# WAL 模式允许读写并发，显著提升性能

# 3. 调整同步模式（牺牲一些安全性换取性能）
conn.execute("PRAGMA synchronous=NORMAL")
# NORMAL 模式在 WAL 下是安全的

# 完整配置示例
conn = sqlite3.connect("langgraph.db", timeout=30)
conn.execute("PRAGMA journal_mode=WAL")
conn.execute("PRAGMA synchronous=NORMAL")
conn.execute("PRAGMA cache_size=-64000")  # 64MB 缓存

checkpointer = SqliteSaver(conn)
```

---

## SqliteSaver vs PostgresSaver

| 特性 | SqliteSaver | PostgresSaver |
|------|-------------|---------------|
| 部署 | 零（文件即可） | 需要 PG 服务 |
| 持久化 | ✅ | ✅ |
| 并发写入 | ⚠️ 有限 | ✅ 高并发 |
| 多进程访问 | ⚠️ 受限 | ✅ 完全支持 |
| 分布式 | ❌ | ✅ |
| 性能 | 中 | 高 |
| 备份 | 简单（复制文件） | 需要工具 |
| 数据量上限 | ~100 GB 理论 | 无实际限制 |
| JSON 查询 | 弱（TEXT 存储） | 强（JSONB） |

### 选择建议

```
数据量小 + 单机 + 低并发？
├── 是 → SqliteSaver（简单够用）
└── 否 → PostgresSaver（生产推荐）

需要多进程/分布式？
├── 是 → PostgresSaver
└── 否 → SqliteSaver 或 PostgresSaver 均可
```

---

## 完整实战示例

```python
import os
import sqlite3
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.sqlite import SqliteSaver

# ===== 状态定义 =====
class ChatState(TypedDict):
    messages: Annotated[list, add_messages]
    user_name: str

# ===== 节点定义 =====
def greet(state: ChatState) -> dict:
    """问候节点"""
    name = state.get("user_name", "用户")
    return {"messages": [("assistant", f"你好，{name}！有什么可以帮你的？")]}

def respond(state: ChatState) -> dict:
    """回复节点"""
    # 可以访问历史消息
    history = state["messages"]
    return {"messages": [("assistant", f"我理解了，之前你说了{len(history)}条消息")]}

# ===== 构建图 =====
graph = StateGraph(ChatState)
graph.add_node("greet", greet)
graph.add_node("respond", respond)
graph.add_edge(START, "greet")
graph.add_edge("greet", "respond")
graph.add_edge("respond", END)

# ===== 配置持久化 =====
db_path = os.path.join(os.path.dirname(__file__), "chat_data.db")
conn = sqlite3.connect(db_path)
conn.execute("PRAGMA journal_mode=WAL")  # 启用 WAL 模式

checkpointer = SqliteSaver(conn)
checkpointer.setup()

app = graph.compile(checkpointer=checkpointer)

# ===== 运行 =====
config = {"configurable": {"thread_id": "user-alice"}}

# 第一次交互
result = app.invoke(
    {"messages": [], "user_name": "Alice"},
    config=config
)
for msg in result["messages"]:
    print(f"[{msg.type}] {msg.content}")

# 第二次交互（历史保留）
result = app.invoke(
    {"messages": [("user", "帮我查一下天气")], "user_name": "Alice"},
    config=config
)
for msg in result["messages"]:
    print(f"[{msg.type}][[ {msg.content}")

# 关闭连接
conn.close()
```

---

## 相关页面

- [[langgraph/modules/langgraph_checkpoint/概述|持久化概述]]
- [[langgraph/modules/langgraph_checkpoint/InMemorySaver|InMemorySaver]]
- [[langgraph/modules/langgraph_checkpoint/PostgresSaver|PostgresSaver]]
- [[langgraph/modules/langgraph_checkpoint/thread_id与会话管理|thread_id 与会话管理]]
