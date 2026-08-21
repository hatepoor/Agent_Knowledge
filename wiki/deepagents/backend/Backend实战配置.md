---
tags: [deepagents, backend, composite, store, sandbox]
created: 2026-07-29
updated: 2026-07-29
type: concept
deepagents_version: ">=0.7"
---

# Backend 实战配置

三个最常见生产场景的完整 Backend 配置代码。

## 一、CompositeBackend — "短期+长期"混合

```python
from deepagents.backends import (
    StateBackend, StoreBackend, CompositeBackend
)
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()  # 生产用 PostgresStore

agent = create_deep_agent(
    model=llm,
    backend=CompositeBackend(
        default=StateBackend(),              # / 路径 = 临时内存
        routes={
            "/memories/": StoreBackend(       # /memories/ = 跨线程持久
                store=store
            ),
            "/cache/": FilesystemBackend(     # /cache/ = 本地磁盘
                root_dir=Path("/tmp/agent-cache")
            ),
        },
    ),
)

# Agent 行为:
# write_file("/report.md")           → StateBackend（线程结束清理）
# write_file("/memories/notes.txt")  → StoreBackend（重启后仍在）
# write_file("/cache/large.csv")     → /tmp/agent-cache/（持久，但可手动清理）
```

## 二、SandboxBackend — 隔离执行

```python
from deepagents.backends.langsmith import LangSmithSandbox
from langsmith.sandbox import SandboxClient

client = SandboxClient()
sandbox = client.create_sandbox(
    image="python:3.11-slim",
    cpu=1, memory_mb=512,
)

agent = create_deep_agent(
    model=llm,
    backend=LangSmithSandbox(sandbox=sandbox),
    # Agent 自动获得 execute 工具
)
```

**自建 Docker 替代**：

```python
# 如果不用 LangSmith，用自建 Docker（参考 wiki/docker/）
from deepagents.backends import FilesystemBackend
from docker_sandbox import DockerSandbox

# DeepAgents 的 SandboxBackend 也可用自定义后端
# 实现 BackendProtocol 接口即可
```

## 三、纯 StoreBackend — 全部持久化

```python
from deepagents.backends import StoreBackend
from langgraph.store.postgres import PostgresStore

store = PostgresStore.from_conn_string(
    "postgresql://user:pass@localhost/agent_db"
)

agent = create_deep_agent(
    model=llm,
    backend=StoreBackend(store=store),
    # 所有文件操作都走 Postgres
    # 重启后文件仍在，不同 thread 可共享
)
```

## 四、相关页面

- [[deepagents/backend/后端概述]] — 6 种 Backend 详解
- [[docker/Docker沙箱概述]] — 自建沙箱与 DeepAgents 沙箱对比
