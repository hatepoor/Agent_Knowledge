---
tags: [langchain, memory, checkpointer, postgres, storage, backend]
created: 2026-07-08
updated: 2026-07-08
type: concept
---

# Checkpointer 存储后端

## 后端对比

| 后端 | 存储位置 | 内存占用 | 持久化 | 适用场景 |
|------|---------|---------|--------|----------|
| InMemorySaver | 进程内存 | 低 | 否 | 开发测试 |
| PostgresSaver | PostgreSQL | 数据库管理 | 是 | 生产部署 |
| SqliteSaver | SQLite 文件 | 文件系统 | 是 | 单机持久化 |
| AzureCosmosDBSaver | Azure Cosmos DB | 云托管 | 是 | Azure 云 |

## 为什么数据库存储不会导致进程 OOM

核心区别：数据**不在 Agent 进程中管理**，而是由独立的数据库服务管理。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           内存存储 vs 数据库存储                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ■ 内存存储                            ■ 数据库存储                         │
│                                                                             │
│    Agent 进程                           Agent 进程                          │
│    ┌──────────────────────┐            ┌──────────────────────────┐         │
│    │  messages: [...]     │            │  SELECT * FROM           │         │
│    │  InMemorySaver       │            │  checkpoints             │         │
│    │  (全量数据在进程内存) │            │  WHERE thread=1;         │         │
│    │  → 进程内存持续增长   │            │  (只在查询时传输)         │         │
│    └──────────────────────┘            └──────────────────────────┘         │
│              ↓                                       ↓                      │
│        进程内存持续增长                      PostgreSQL 进程                  │
│              ↓                              ┌──────────────────────┐         │
│       ⚠️ OOM 风险                          │  checkpoints 表      │         │
│                                           │  stores 表           │         │
│                                           │  (数据在磁盘)         │         │
│                                           └──────────────────────┘         │
│                                                    ↓                       │
│                                          内存由数据库管理                   │
│                                          ✅ Agent OOM 风险消除              │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  关键结论：内存存储在 Agent 进程内保存全量数据，易导致 OOM；                │
│           数据库存储将数据放在磁盘，由数据库管理内存，消除 OOM 风险。        │
└─────────────────────────────────────────────────────────────────────────────┘
```

Agent 每次 invoke 时，从数据库**按需拉取**当前 thread 的最新状态，执行完后把新状态**写回**数据库。Agent 进程内存里只保留**当前这一轮**的数据，不会累积历史。

| 风险 | 内存存储 | 数据库存储 |
|------|---------|-----------|
| **Agent 进程 OOM** | 🔴 对话越长，进程内存越大 | ✅ 进程内存几乎恒定 |
| **数据库磁盘满** | — | ⚠️ 磁盘满会报错，但不影响 Agent 进程 |

```
# 1000 轮对话后的内存占用对比
# ===== 内存存储 =====
# InMemorySaver 将所有消息保存在 Python 进程内的 dict 对象中
# 1000 轮 ≈ 2000 条消息（每轮含 user + assistant 各一条）
# 每条消息平均约 150KB（含 role、content、额外元数据等）
InMemorySaver → 全部 1000 轮消息在 dict 中 → ~300 MB
# 风险：进程内存持续增长，最终可能触发 OOM Killer

# ===== 数据库存储 =====
# PostgresSaver 只在需要时从数据库拉取当前轮所需的状态
# 进程内存中只保留最近一轮的消息数据（约 1MB 以内）
# 其余 999 轮的历史数据全部存储在数据库磁盘上，不占用进程内存
PostgresSaver → 只拉取最新状态 → ~1 MB（其余在数据库磁盘上）
# 优势：进程内存几乎恒定，不受对话长度影响
```

## PostgresSaver 的表结构（短期记忆）

调用 `cp.setup()` 时自动创建，核心是 `checkpoints` 表：

```sql
-- checkpoints 表：存储 Agent 每一步执行后的完整状态快照
-- 调用 checkpointer.setup() 时自动在 PostgreSQL 中创建此表
CREATE TABLE checkpoints (
    -- thread_id: 对话线程的唯一标识，同一个对话的所有检查点共享同一个 thread_id
    -- 用于区分不同用户/不同对话会话的数据
    thread_id     TEXT NOT NULL,

    -- checkpoint_id: 当前检查点的唯一标识，通常按步骤递增（如 step-1, step-2）
    -- 与 thread_id 组合构成主键，保证同一线程内每个检查点唯一
    checkpoint_id TEXT NOT NULL,

    -- parent_id: 指向该检查点的上一个检查点 ID，形成链式结构
    -- 用途：支持"时间旅行"（time travel）功能，可以回溯到任意历史步骤
    -- 第一个检查点的 parent_id 为 NULL（没有父节点）
    parent_id     TEXT,

    -- checkpoint: 核心字段 —— 以 JSONB 格式存储完整的 Agent 状态快照
    -- 包含 messages 列表（所有对话消息）以及自定义的状态字段
    -- JSONB 类型支持高效的 JSON 查询和部分更新，无需解析整个文档
    checkpoint    JSONB NOT NULL,

    -- metadata: 元数据字段，存储检查点的辅助信息
    -- 典型内容：{"step": 1, "source": "loop", "model": "gpt-5.5"} 等
    -- 用于调试、审计和按条件筛选检查点
    metadata      JSONB NOT NULL,

    -- created_at: 检查点的创建时间戳，TIMESTAMPTZ 带时区信息
    -- DEFAULT NOW() 表示插入时自动记录当前时间，无需手动传入
    created_at    TIMESTAMPTZ DEFAULT NOW(),

    -- 复合主键：(thread_id, checkpoint_id) 确保同一线程内检查点 ID 不重复
    -- 这也是 PostgreSQL 聚簇索引的物理排序依据
    PRIMARY KEY (thread_id, checkpoint_id)
);

-- 为 thread_id 创建普通索引，加速按对话线程查询检查点的操作
-- 典型场景：恢复对话时通过 WHERE thread_id = 'xxx' 快速定位该线程的所有检查点
-- 没有此索引的话，查询需要全表扫描，对话量大时性能会急剧下降
CREATE INDEX idx_checkpoints_thread ON checkpoints(thread_id);
```

每次 Agent 步骤的写入过程：

```sql
-- ===== 写入检查点 =====
-- 每执行一步 Agent 循环，就向 checkpoints 表插入一条新记录
-- 注意：这里不会 UPDATE 已有记录，而是每次都 INSERT 新行（追加式写入）
-- 好处：保留完整历史，支持时间旅行回溯

-- 第 1 轮：用户发送第一条消息，Agent 处理后保存状态
INSERT INTO checkpoints (thread_id, checkpoint_id, checkpoint, metadata)
VALUES (
  'thread-001',  -- 对话线程 ID，标识这是哪个对话
  'step-1',      -- 检查点 ID，按步骤递增命名
  -- checkpoint 字段：JSONB 格式，包含当前所有消息的完整快照
  '{"messages": [{"role": "user", "content": "Hi"}]}',
  -- metadata 字段：记录这是第 1 步，方便调试和查询
  '{"step": 1}'
);

-- 第 2 轮：Agent 回复后保存新状态，parent_id 指向上一步（step-1）
INSERT INTO checkpoints (thread_id, checkpoint_id, parent_id, checkpoint, metadata)
VALUES (
  'thread-001',  -- 同一个对话线程
  'step-2',      -- 新的检查点 ID
  'step-1',      -- parent_id 形成链式结构：step-2 → step-1
  -- checkpoint 包含截至目前的完整消息历史（用户消息 + AI 回复）
  -- 注意：是完整快照，不是增量！每一步都存完整的 messages 列表
  '{"messages": [{"role": "user", "content": "Hi"}, {"role": "assistant", "content": "Hello"}]}',
  '{"step": 2}'
);

-- ===== 恢复对话时：查询最新检查点 =====
-- 当用户重新连接或继续对话时，需要恢复之前的对话状态
-- 按 checkpoint_id 降序排列，取最新的一条记录
-- ORDER BY + LIMIT 1 比 MAX() 更高效，因为可以直接利用索引
SELECT checkpoint FROM checkpoints
WHERE thread_id = 'thread-001'
ORDER BY checkpoint_id DESC LIMIT 1;
-- 返回结果就是最近一次保存的完整状态，Agent 据此继续执行
```

## PostgresStore 的表结构（长期记忆）

键值对风格，通过 namespace 做多租户隔离：

```sql
-- store 表：长期记忆的键值对存储，用于保存跨对话的持久化信息
-- 与 checkpoints 表（短期记忆）不同，store 的数据不会随对话结束而清理
-- 典型用途：用户偏好、知识片段、自定义长期记忆条目

CREATE TABLE store (
    -- namespace: 命名空间，使用 PostgreSQL 的 TEXT[] 数组类型
    -- 作用：实现多租户隔离，不同用户/不同类别的记忆互不干扰
    -- 示例：{"prefs", "alice"} 表示用户 alice 的偏好设置命名空间
    --        {"docs", "project_x"} 表示 project_x 的文档命名空间
    namespace   TEXT[] NOT NULL,

    -- key: 键名，在同一个 namespace 内唯一标识一条记忆
    -- 示例："theme"、"favorite_color"、"user_bio" 等
    key         TEXT NOT NULL,

    -- value: 记忆的值，JSONB 格式存储，支持任意结构的 JSON 数据
    -- 示例：{"value": "dark"}、{"value": "用户喜欢Python和机器学习"}
    -- JSONB 类型支持 GIN 索引和 JSON 路径查询
    value       JSONB NOT NULL,

    -- created_at: 首次创建时间，插入时自动记录
    created_at  TIMESTAMPTZ DEFAULT NOW(),

    -- updated_at: 最后更新时间，每次 upsert 时刷新为当前时间
    -- 用于追踪记忆的新鲜度，也可用于清理过期数据
    updated_at  TIMESTAMPTZ DEFAULT NOW(),

    -- 复合主键：(namespace, key) 保证同一命名空间内键名不重复
    -- PostgreSQL 对数组类型的主键支持良好
    PRIMARY KEY (namespace, key)
);

-- pgvector 扩展的向量索引（仅在启用语义检索时需要）
-- 作用：加速基于 embedding 相似度的记忆检索查询
-- 使用 IVFFlat 算法：将向量空间划分为多个倒排文件，牺牲少量精度换取大幅查询加速
-- vector_cosine_ops：使用余弦相似度作为距离度量，适合文本语义匹配
-- WHERE embedding IS NOT NULL：部分索引，只对有 embedding 的行建索引
--   节省存储空间，避免为不需要语义检索的普通键值对维护向量索引
CREATE INDEX idx_store_embedding ON store
USING ivfflat (embedding vector_cosine_ops)
WHERE embedding IS NOT NULL;
```

读写操作：

```sql
-- ===== 写入（Upsert 模式）=====
-- 使用 INSERT ... ON CONFLICT 实现"有则更新，无则插入"的语义
-- 这是 PostgreSQL 的 UPSERT 语法，避免先查再写的竞态条件
INSERT INTO store (namespace, key, value)
VALUES (
  '{"prefs", "alice"}',  -- 命名空间：用户 alice 的偏好设置
  'theme',               -- 键名：主题偏好
  '{"value": "dark"}'    -- 值：深色模式
)
-- ON CONFLICT：当 (namespace, key) 已存在时触发冲突处理
-- 而不是报错，实现幂等写入（多次写入同一键不会出错）
ON CONFLICT (namespace, key) DO UPDATE
SET
  value = EXCLUDED.value,     -- EXCLUDED 代表本次 INSERT 试图写入的新值
  updated_at = NOW();         -- 同时刷新更新时间戳

-- ===== 精确读取 =====
-- 通过 namespace + key 精确定位一条记忆，O(1) 复杂度的主键查询
SELECT value FROM store
WHERE namespace = '{"prefs", "alice"}' AND key = 'theme';
-- 返回：{"value": "dark"}

-- ===== 语义检索（配合 pgvector 扩展）=====
-- 当需要模糊匹配或"相似记忆"搜索时，使用向量相似度检索
-- '<=>' 是 pgvector 提供的余弦距离运算符，值越小越相似
-- '[0.1, 0.2, ...]' 是查询文本的 embedding 向量（由 embedding 模型生成）
-- LIMIT 5：返回最相似的前 5 条记忆
-- 典型场景：用户问了一个问题，从长期记忆中检索语义最相关的知识片段
SELECT key, value FROM store
WHERE namespace = '{"docs"}'
ORDER BY embedding <=> '[0.1, 0.2, ...][['
LIMIT 5;
```

## 常见误解

| 误解 | 正解 |
|------|------|
| "数据库存就不会 OOM" | 是**进程**不会 OOM，但数据库磁盘满了照样报错 |
| "数据库慢" | 读写在毫秒级，对 LLM 调用（秒级）来说可以忽略 |
| "数据库存的数据格式不同" | 存的仍然是同样的 messages JSON，只是放在数据库里 |
| "用了数据库就不需要裁剪消息了" | 仍需裁剪——不裁剪会导致数据库表膨胀、查询变慢 |

## 相关页面

- [[langchain/memory/短期记忆]] — 短期记忆核心概念与用法
- [[langchain/memory/记忆系统性能]] — 内存估算、OOM 预防与性能开销
- [[langchain/modules/langchain_stores/概述]] — Store 长期记忆模块
- [[langchain/modules/langchain_checkpoints/概述]] — Checkpoint 模块
