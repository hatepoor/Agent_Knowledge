---
tags: [langchain, stores, api, key-value]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# Store 核心 API

Store 提供了一组简洁的键值存储 API，所有操作都基于 namespace（命名空间）进行数据隔离。

## API 一览

| 方法 | 作用 |
|------|------|
| `get(namespace, key)` | 读取 |
| `put(namespace, key, value)` | 写入/更新 |
| `delete(namespace, key)` | 删除 |
| `search(namespace, query=..., limit=...)` | 检索（语义或前缀） |
| `batch(...)` | 批量操作 |

## get -- 读取

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
namespace = ("users", "user-123")

store.put(namespace, "preference_theme", {"value": "dark"})

# 读取
item = store.get(namespace, "preference_theme")
print(item.value)  # {"value": "dark"}
```

## put -- 写入/更新

```python
namespace = ("users", "user-123")

# 写入
store.put(namespace, "preference_theme", {"value": "dark"})
store.put(namespace, "preference_language", {"value": "Chinese"})

# 更新（同 key 覆盖）
store.put(namespace, "preference_theme", {"value": "light"})
```

## delete -- 删除

```python
store.delete(namespace, "preference_theme")
```

## search -- 检索

```python
# 列出命名空间下所有键
items = store.search(namespace)
for item in items:
    print(f"  {item.key}: {item.value}")
```

### 语义检索（PostgresStore）

```python
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://user:pass@localhost:5432/agent_db"
embeddings = ...  # 你的 Embeddings 实例

with PostgresStore.from_conn_string(DB_URI, index={"dims": 1536, "embed": embeddings}) as store:
    store.setup()

    # 写入时指定索引字段
    store.put(("docs",), "doc1", {"text": "LangChain tutorial content"}, index=["text"][[)

    # 语义检索
    results = store.search(("docs",), query="LangChain tutorial", limit=5)
```

## 与 Agent 工具集成

Store 通常通过 `runtime.store` 在工具中使用：

```python
from langchain.tools import tool, ToolRuntime

@tool
def save_user_pref(key: str, value: str, runtime: ToolRuntime) -> str:
    """Save user preference."""
    ns = ("prefs", runtime.context.user_id)
    runtime.store.put(ns, key, {"value": value, "ts": "2026-07-07"})
    return f"Saved {key}={value}"

@tool
def read_user_pref(key: str, runtime: ToolRuntime) -> str:
    """Read user preference."""
    ns = ("prefs", runtime.context.user_id)
    item = runtime.store.get(ns, key)
    return str(item.value) if item else "Not set"
```

## 注意事项

- `value` 必须是 JSON 可序列化的字典
- namespace 是元组，用逗号分隔各层级
- `search` 的行为取决于实现：InMemoryStore 返回所有条目，PostgresStore 支持语义搜索
- 向量检索需在 `put` 时指定 `index` 字段名列表

## 相关

- [[langchain/modules/langchain_stores/概述|<- 返回 stores 概述]]
- [[langchain/modules/langchain_stores/命名空间设计|命名空间设计]]
