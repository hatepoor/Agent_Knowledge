---
tags: [interview, postgresql, pgvector, RAG, AI, 面试]
type: guide
created: 2026-10-18
updated: 2026-10-18
---

# PostgreSQL 在 AI 应用中的实践

> **适用场景**：Agent / RAG 岗的加分题。面试官想确认：你是「拿 PG 当向量库用」，还是真的理解它的边界。
> **主线**：**PG 的优势不是「向量检索最快」，而是「向量、元数据、事务、业务数据在同一个库里」。**

---

## Q1. 为什么 RAG / Agent 项目常用 PostgreSQL，而不是直接上专用向量库？

**答：**

因为真实项目需要的**不只是向量检索**：

| 需求 | 专用向量库 | PostgreSQL + pgvector |
|------|-----------|----------------------|
| 向量相似度检索 | ✅ 强 | ✅ 够用 |
| **元数据过滤**（租户、权限、时间、状态） | 支持但表达力有限 | ✅ 完整 SQL |
| **事务**（文档与切片同时写入） | ❌ 多数不支持 | ✅ ACID |
| **与业务表 JOIN** | ❌ 跨系统 | ✅ 同库 JOIN |
| **权限/行级安全** | 另做 | ✅ RLS |
| **运维成熟度** | 新组件，团队要学 | ✅ 已有备份/监控/复制 |
| **规模** | 亿级以上更强 | 千万级以内很稳 |

**一句话**：**PG 是「一个库解决 80% 问题」，专用向量库是「一个点做到极致」。**

**追问：什么时候必须上专用向量库？**
→ 向量规模到**亿级**、QPS 很高、需要极致召回/延迟，或者需要多租户大规模隔离时。此时 PG 的索引构建、内存、 vacuum 都会成为瓶颈。

---

## Q2. pgvector 怎么用？建表和建索引的基本流程是什么？

**答：**

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id        bigserial PRIMARY KEY,
    tenant_id bigint NOT NULL,
    content   text,
    metadata  jsonb,
    embedding vector(1536)          -- 维度必须与模型一致
);

-- HNSW 索引（推荐默认）
CREATE INDEX idx_documents_embedding
ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

查询：

```sql
SELECT id, content, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE tenant_id = 42
ORDER BY embedding <=> $1
LIMIT 10;
```

要点：

| 点 | 说明 |
|----|------|
| **维度固定** | `vector(1536)` 的维度必须和 embedding 模型输出一致，换模型要改列/重建 |
| **距离操作符要和索引 opclass 匹配** | `<=>` 配 `vector_cosine_ops`，`<->` 配 `vector_l2_ops` |
| **`ORDER BY` 必须用距离操作符** | 否则用不上向量索引 |
| **`LIMIT` 必加** | 没有限制会退化为全表计算 |

**追问：`vector` 和 `halfvec` 怎么选？**
→ `halfvec` 是半精度（2 字节/维），索引更小、构建更快，召回损失通常很小。**大表/内存紧张时优先考虑 halfvec**；对精度极敏感的场景用 `vector`。

---

## Q3. 余弦距离、L2 距离、内积，应该选哪个？

**答：**

| 操作符 | 距离 | 适用 |
|--------|------|------|
| `<=>` | 余弦距离（1 - cosine similarity） | **文本 embedding 默认**，只关心方向 |
| `<->` | 欧氏距离（L2） | 向量已归一化时与余弦等价；图像/坐标类常用 |
| `<#>` | 负内积 | 检索「最大内积」，推荐系统常见 |

选择原则：

1. **看模型训练时用的度量**。大多数文本 embedding（OpenAI、BGE 等）用余弦相似度训练，检索就用余弦；
2. 如果向量**已经 L2 归一化**，余弦距离和 L2 距离排序**等价**，可以选索引更快的；
3. **不要混用**：建索引用的 opclass 和查询用的操作符必须一致，否则索引不生效。

**追问：为什么归一化后余弦和 L2 等价？**
→ 因为 `||a-b||² = ||a||² + ||b||² - 2a·b`。归一化后 `||a||=||b||=1`，所以 L2 距离只取决于内积，排序结果和余弦一致。

---

## Q4. HNSW 和 IVFFlat 有什么区别？参数怎么调？

**答：**

| 维度 | HNSW | IVFFlat |
|------|------|---------|
| **结构** | 多层图，逐层跳转 | 聚类成 `lists` 个桶，查询扫 `probes` 个桶 |
| **构建** | 慢、占内存 | 快、占内存少 |
| **查询** | 快、召回高 | 较快，依赖 `probes` |
| **增量插入** | ✅ 友好 | ⚠️ 数据分布变化后要重建 |
| **建索引前提** | 可空表建 | **建议先有数据再建**（要先聚类） |
| **参数** | `m`、`ef_construction`、`ef_search` | `lists`、`probes` |

**HNSW 参数**：

| 参数 | 作用 | 调大 |
|------|------|------|
| `m` | 每个节点的邻居数 | 召回↑、内存↑、构建慢 |
| `ef_construction` | 构建时的候选集大小 | 索引质量↑、构建慢 |
| `ef_search` | 查询时的候选集大小（**运行时设**） | 召回↑、延迟↑ |

```sql
SET hnsw.ef_search = 40;   -- 默认 40，按召回要求调
```

**IVFFlat 参数**：

| 参数 | 建议 |
|------|------|
| `lists` | 行数 < 100 万用 `rows/1000`，超过用 `sqrt(rows)` |
| `probes` | 从 `sqrt(lists)` 起步，按召回调 |

**追问：为什么 IVFFlat 要在有数据后建？**
→ 它要先对现有向量做 k-means 聚类，数据为空或很少时聚类没有意义，索引质量差。**HNSW 没有这个限制，可以空表建、持续插入。**

---

## Q5. 向量检索和关键词检索怎么结合？混合检索怎么做？

**答：**

三种融合方式，从简到繁：

**① 两路召回 + 应用层融合（最常用）**

```sql
-- 向量召回
SELECT id, 1 - (embedding <=> $1) AS score FROM docs
WHERE tenant_id = 42 ORDER BY embedding <=> $1 LIMIT 20;

-- 关键词召回（全文索引）
SELECT id, ts_rank(tsv, query) AS score FROM docs, to_tsquery('simple', $2) query
WHERE tsv @@ query AND tenant_id = 42 LIMIT 20;
```

再用 **RRF（Reciprocal Rank Fusion）** 合并：

```
score(doc) = Σ 1 / (k + rank_i(doc))     # k 常取 60
```

RRF 只看排名、不看分数尺度，**避免向量相似度和 BM25 分数不可比的问题**。

**② SQL 里一次完成**

用 CTE 把两路召回 `UNION ALL`，再按 `id` 聚合分数。适合中等数据量。

**③ 先过滤再检索**

`WHERE tenant_id = 42 AND status = 'active'` 先缩小范围，再做向量排序。注意**过滤太狠会导致 HNSW 召回下降**（图里可走的邻居变少）。

**追问：为什么需要混合检索？**
→ 向量擅长**语义相似**（「退款」≈「退货」），关键词擅长**精确匹配**（订单号、错误码、专有名词）。**单靠向量会漏掉精确查询，单靠关键词会漏掉同义表达。**

---

## Q6. RAG 场景下，PG 的表结构怎么设计？

**答：**

一个可落地的 schema：

```sql
CREATE TABLE documents (
    id          bigserial PRIMARY KEY,
    tenant_id   bigint NOT NULL,
    source      text,                 -- 文件路径/URL
    title       text,
    metadata    jsonb,                -- 页码、章节、权限标签
    created_at  timestamptz DEFAULT now()
);

CREATE TABLE chunks (
    id           bigserial PRIMARY KEY,
    document_id  bigint NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    tenant_id    bigint NOT NULL,     -- 冗余，避免每次 JOIN
    chunk_index  int NOT NULL,
    content      text NOT NULL,
    token_count  int,
    embedding    vector(1024),
    tsv          tsvector GENERATED ALWAYS AS (to_tsvector('simple', content)) STORED
);

CREATE INDEX ON chunks (tenant_id, document_id);
CREATE INDEX ON chunks USING gin (tsv);
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops);
```

设计要点：

| 点 | 原因 |
|----|------|
| **`tenant_id` 冗余到 chunks** | 检索时直接过滤，避免 JOIN 文档表 |
| **`ON DELETE CASCADE`** | 删文档自动清切片，避免孤儿数据 |
| **`tsv` 生成列** | 关键词检索和向量检索在同一张表 |
| **`metadata` 用 JSONB + GIN** | 灵活的权限/标签过滤 |
| **`chunk_index`** | 召回后按原文顺序拼接上下文 |

**追问：为什么不在应用层拼上下文，而非要在 SQL 里做？**
→ 因为**过滤和排序要在数据库完成**，否则要把大量候选拉到应用层再筛，网络和内存成本高。SQL 的优势就是「过滤 + 排序 + 限制」一次完成。

---

## Q7. 向量数据量大了，pgvector 性能怎么保证？

**答：**

六个方向：

| 方向 | 做法 |
|------|------|
| **索引** | HNSW 优先；大表用 `halfvec` 降内存；定期 `REINDEX` |
| **分区** | 按 `tenant_id` 或时间分区，缩小单索引规模 |
| **过滤前置** | 先按租户/时间缩小范围，再做向量排序 |
| **减少维度** | 换更小的 embedding 模型，或 PCA 降维（需评估召回损失） |
| **参数调优** | `maintenance_work_mem` 加速建索引；`hnsw.ef_search` 平衡召回与延迟 |
| **冷热分离** | 老数据归档到单独表/分区，主表只留热数据 |

**建索引时的注意点**：

- HNSW 构建**很吃内存**，`maintenance_work_mem` 太小会落盘，构建时间成倍增加；
- 大表建索引建议低峰期，或先用 `CREATE INDEX CONCURRENTLY`（不阻塞写，但更慢）；
- 建完后 `ANALYZE`，让优化器知道数据分布。

**追问：为什么加了 `WHERE tenant_id = ?` 之后召回变差了？**
→ HNSW 是近似检索，过滤条件会让图遍历的可达节点变少，可能提前终止。解决办法：① 按租户分区，让每个索引只含一个租户；② 提高 `ef_search`；③ 先过滤再精确计算（候选集不大时）。

---

## Q8. LangGraph 的 `PostgresSaver` 在生产上要注意什么？

**答：**

它是「用 PG 存 Agent 状态」，压力模型和业务库不同：

| 注意点 | 说明 |
|--------|------|
| **表会快速增长** | 每个 superstep 一个 checkpoint，长会话几千条很常见 |
| **单行可能很大** | State 里有 messages、检索结果，JSONB 可能几十 KB |
| **写入频率高** | 每步都写，PG 的 WAL 和 autovacuum 压力明显 |
| **`thread_id` 不宜过长** | 不要塞长 JSON；理论上 TEXT 无硬限制，但建议控制在 255 字符内 |
| **`setup()` 要执行** | 否则表不存在；升级后可能要重跑 |
| **连接池** | 用 `AsyncPostgresSaver` + pool，不要每请求新建连接 |
| **多租户隔离** | 独立 schema 或独立库，便于按租户清理 |

优化手段：

1. **State 瘦身**：大对象外置，State 只存 id；
2. **消息裁剪/摘要**：别让 messages 无限增长；
3. **定期归档**：已结束会话的 checkpoint 迁冷存储；
4. **按 `thread_id` 清理**：整条时间线删，不要删中间；
5. **监控表大小和写入延迟**：`pg_total_relation_size('checkpoints')`。

**追问：Checkpointer 的表和业务表放同一个库吗？**
→ 可以，但要注意**资源隔离**：Agent 的高频写入可能影响业务查询。规模大了建议独立实例/独立 schema，至少给不同的连接池和资源组。

---

## Q9. PG + pgvector 和 Milvus / Qdrant 这类专用向量库怎么选？

**答：**

| 维度 | PG + pgvector | 专用向量库 |
|------|--------------|-----------|
| **规模** | 百万~千万级表现好 | 亿级以上更强 |
| **功能完整度** | SQL、事务、JOIN、权限 | 向量检索功能更深（多副本、量化、分布式） |
| **运维** | 复用现有 PG 体系 | 新组件、新团队技能 |
| **一致性** | 强一致，事务内写入 | 多数最终一致 |
| **延迟** | 中等 | 低（专用索引/分布式） |
| **成本** | 低（复用现有实例） | 高（新集群） |
| **适用** | 业务数据 + 向量同库、中等规模 | 纯向量检索、超大规模、高 QPS |

**决策建议**：

```
数据量 < 1000 万 且 需要和业务数据 JOIN/事务  → PG
数据量 > 1 亿 或 QPS 很高 或 需要多副本分布式   → 专用向量库
介于两者之间                                   → 先 PG，压测后再决定
```

**追问：能不能两个都用？**
→ 能，常见架构是「PG 存业务数据和元数据，向量库存 embedding」，检索时先向量库召回 id，再回 PG 取详情。代价是**两套存储的一致性和运维复杂度**，中小团队通常不值得。

**追问：面试官问「你们的 RAG 为什么选 PG」，怎么答最有说服力？**
→ 不要只说「够用」。要说：**「我们的数据量和租户规模在千万级以内，且需要按权限过滤、和业务表 JOIN、保证文档与切片的事务一致；专用向量库在这些点上要额外做同步和补偿，复杂度不划算。我们压测过 P99，在目标 QPS 下满足要求。」**——有边界、有对比、有数据。

---

## 本篇自测

- [ ] PG 做 RAG 的四个优势，什么时候必须上专用向量库
- [ ] pgvector 的建表/索引/查询三件套，维度与 opclass 的约束
- [ ] 三种距离度量的选择依据，归一化后为什么等价
- [ ] HNSW 与 IVFFlat 的六维对比，各自参数怎么调
- [ ] 混合检索的 RRF 公式，为什么用排名而不是分数
- [ ] RAG schema 的五个设计要点
- [ ] 向量规模变大后的六个优化方向
- [ ] `PostgresSaver` 的七个注意点
- [ ] PG 与专用向量库的选型边界，怎么用「边界 + 数据」回答

---

## 导航

[[postgresql/interview/README|PostgreSQL 面试题库首页]] · 上一篇：[[postgresql/interview/07-复制备份与高可用]] · 相关：[[rag/README|RAG 知识库]] · [[langgraph/interview/04-持久化与Checkpointer]]
