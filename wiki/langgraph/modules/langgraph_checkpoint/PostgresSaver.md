---
tags: [langgraph, checkpoint, PostgresSaver, PostgreSQL, 生产环境, 持久化]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# PostgresSaver（PostgreSQL 检查点器）

## 概述

`PostgresSaver` 是基于 PostgreSQL 的 Checkpointer 实现，适用于生产环境。它提供持久化存储、高并发支持、水平扩展能力，是 LangGraph 生产部署的推荐方案。

### 特点

- **持久化**：数据存储在 PostgreSQL 中，进程重启不丢失
- **高并发**：支持多进程、多实例同时读写
- **可靠性**：利用 PostgreSQL 的事务和 WAL 机制保证数据一致性
- **可扩展**：可以利用 PostgreSQL 的复制和分区功能扩展

### 适用场景

- 生产环境部署
- 多进程/多实例应用
- 需要数据持久化的场景
- 高并发聊天应用

---

## 安装

```bash
# 安装 langgraph-postgres 包
pip install langgraph-checkpoint-postgres

# 还需要安装 psycopg（PostgreSQL 驱动）
# 方式一：psycopg3（推荐）
pip install "psycopg[binary]"

# 方式二：psycopg2
pip install "psycopg2-binary"
```

---

## 导入与创建

```python
from langgraph.checkpoint.postgres import PostgresSaver

# 方式一：使用连接字符串
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:password@localhost:5432/langgraph_db"
)

# 方式二：使用已有的连接对象（推荐用于生产）
import psycopg
conn = psycopg.connect("postgresql://user:password@localhost:5432/langgraph_db")
checkpointer = PostgresSaver(conn)
```

---

## 基本用法

### 使用上下文管理器（推荐）

```python
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.graph import StateGraph, MessagesState, START, END

# 构建图
graph = StateGraph(MessagesState)
graph.add_node("chatbot", chatbot_node)
graph.add_edge(START, "chatbot")
graph.add_edge("chatbot", END)

# 使用上下文管理器确保连接正确关闭
with PostgresSaver.from_conn_string(
    "postgresql://user:password@localhost:5432/langgraph_db"
) as checkpointer:
    # 创建表（首次使用时需要调用）
    checkpointer.setup()

    # 编译图
    app = graph.compile(checkpointer=checkpointer)

    # 多轮对话
    config = {"configurable": {"thread_id": "user-123"}}

    # 第一轮
    result = app.invoke(
        {"messages": [("user", "你好")]},
        config=config
    )

    # 第二轮
    result = app.invoke(
        {"messages": [("user", "我叫小明")]},
        config=config
    )
# 退出 with 块后，连接自动关闭
```

### 使用异步上下文管理器

```python
import asyncio
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

async def main():
    async with AsyncPostgresSaver.from_conn_string(
        "postgresql://user:password@localhost:5432/langgraph_db"
    ) as checkpointer:
        # 创建表
        await checkpointer.setup()

        # 编译图
        app = graph.compile(checkpointer=checkpointer)

        # 运行
        config = {"configurable": {"thread_id": "async-user-123"}}
        result = await app.ainvoke(
            {"messages": [("user", "你好")]},
            config=config
        )
        print(result)

asyncio.run(main())
```

---

## setup() 方法

`setup()` 方法创建 LangGraph 所需的数据库表。**首次使用前必须调用一次**。

```python
# 创建表
checkpointer.setup()
```

### 创建的表结构

`PostgresSaver` 会创建以下表：

#### 1. checkpoints 表

存储检查点的主数据：

```sql
CREATE TABLE checkpoints (
    thread_id TEXT NOT NULL,         -- 会话线程ID
    checkpoint_id UUID NOT NULL,     -- 检查点唯一ID
    parent_checkpoint_id UUID,       -- 父检查点ID（形成链）
    checkpoint_ns TEXT DEFAULT '',   -- 命名空间（子图用）
    type TEXT,                       -- 检查点类型
    checkpoint_json JSONB NOT NULL,  -- 检查点数据（JSON格式）
    metadata JSONB DEFAULT '{}',     -- 元数据
    created_at TIMESTAMP DEFAULT NOW(),  -- 创建时间
    PRIMARY KEY (thread_id, checkpoint_id, checkpoint_ns)
);

-- 索引
CREATE INDEX idx_checkpoints_thread ON checkpoints (thread_id);
CREATE INDEX idx_checkpoints_created ON checkpoints (created_at);
```

#### 2. checkpoint_writes 表

存储节点的中间写入（用于崩溃恢复）：

```sql
CREATE TABLE checkpoint_writes (
    thread_id TEXT NOT NULL,         -- 会话线程ID
    checkpoint_id UUID NOT NULL,     -- 关联的检查点ID
    checkpoint_ns TEXT DEFAULT '',   -- 命名空间
    task_id UUID NOT NULL,           -- 任务ID
    idx INTEGER NOT NULL,            -- 写入顺序
    channel TEXT NOT NULL,           -- 状态字段名
    type TEXT,                       -- 数据类型
    blob BYTEA NOT NULL,             -- 写入数据
    PRIMARY KEY (thread_id, checkpoint_id, checkpoint_ns, task_id, idx)
);
```

---

## 连接字符串配置

### 基本格式

```
postgresql://[user]:[password]@[host]:[port]/[database]
```

### 常见配置示例

```python
# 本地开发
conn_string = "postgresql://postgres:password@localhost:5432/langgraph"

# 带 SSL
conn_string = "postgresql://user:pass@host:5432/db?sslmode=require"

# 带连接池参数
conn_string = "postgresql://user:pass@host:5432/db?connect_timeout=10"

# Supabase 示例
conn_string = "postgresql://postgres:[password]@db.[project].supabase.co:5432/postgres"

# Neon Serverless 示例
conn_string = "postgresql://user:pass@host.neon.tech/db?sslmode=require"
```

### 环境变量配置（推荐）

```python
import os
from langgraph.checkpoint.postgres import PostgresSaver

# 从环境变量读取连接字符串
conn_string = os.environ.get("DATABASE_URL", "postgresql://localhost:5432/langgraph")

with PostgresSaver.from_conn_string(conn_string) as checkpointer:
    checkpointer.setup()
    app = graph.compile(checkpointer=checkpointer)
    # ...
```

---

## 连接池配置

对于生产环境，建议使用连接池：

```python
from langgraph.checkpoint.postgres import PostgresSaver
import psycopg

# 使用连接池
conn_pool = psycopg.ConnectionPool(
    conninfo="postgresql://user:pass@host:5432/db",
    min_size=5,    # 最小连接数
    max_size=20,   # 最大连接数
    timeout=30,    # 获取连接超时时间
)

# 从连接池创建 checkpointer
checkpointer = PostgresSaver(conn_pool)
checkpointer.setup()

app = graph.compile(checkpointer=checkpointer)
```

---

## 性能特性

### 读写性能

| 操作 | 性能 | 说明 |
|------|------|------|
| 写入检查点 | ~5-20ms | 取决于状态大小和网络延迟 |
| 读取最新检查点 | ~2-10ms | 有索引优化 |
| 读取历史检查点 | ~2-10ms | 按主键查询 |
| 列出所有线程 | ~10-50ms | 取决于数据量 |

### 容量规划

```
单个检查点大小 ≈ 状态序列化大小 + 元数据（~1-2 KB）

典型场景：
- 简单聊天（10条消息）：~5-15 KB/检查点
- 复杂 RAG（含文档）：~50-200 KB/检查点
- 每轮对话检查点数 ≈ 节点数 + 1

示例估算：
- 1000 个并发用户
- 每人 20 轮对话
- 每轮 3 个检查点
- 每个检查点 10 KB

总存储 ≈ 1000 × 20 × 3 × 10 KB ≈ 600 MB
```

### 性能优化建议

```python
# 1. 使用连接池（避免频繁创建/关闭连接）
# 2. 控制状态大小（避免在状态中存储大文档）
# 3. 定期清理旧检查点
# 4. 使用合适的 PostgreSQL 配置（shared_buffers, work_mem 等）

# 清理旧检查点的 SQL 示例（手动执行）
# DELETE FROM checkpoints
# WHERE created_at < NOW() - INTERVAL '30 days';
```

---

## 错误处理

```python
from langgraph.checkpoint.postgres import PostgresSaver

try:
    with PostgresSaver.from_conn_string(conn_string) as checkpointer:
        checkpointer.setup()
        app = graph.compile(checkpointer=checkpointer)
        result = app.invoke(input, config=config)
except psycopg.OperationalError as e:
    print(f"数据库连接错误: {e}")
    # 处理连接失败（重试、降级等）
except psycopg.Error as e:
    print(f"数据库操作错误: {e}")
    # 处理查询错误
```

---

## 与 InMemorySaver 的迁移

```python
# 开发阶段：使用 InMemorySaver
from langgraph.checkpoint.memory import MemorySaver
checkpointer = MemorySaver()

# 生产阶段：只需替换 checkpointer，其他代码不变
from langgraph.checkpoint.postgres import PostgresSaver
with PostgresSaver.from_conn_string(DB_URL) as checkpointer:
    checkpointer.setup()
    app = graph.compile(checkpointer=checkpointer)
    # 图的定义和节点代码完全不需要修改！
```

---

## 相关页面

- [[langgraph/modules/langgraph_checkpoint/概述|持久化概述]]
- [[langgraph/modules/langgraph_checkpoint/InMemorySaver|InMemorySaver]]
- [[langgraph/modules/langgraph_checkpoint/SqliteSaver|SqliteSaver]]
- [[langgraph/modules/langgraph_checkpoint/thread_id与会话管理|thread_id 与会话管理]]
