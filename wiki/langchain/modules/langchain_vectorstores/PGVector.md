---
tags: [langchain, vector-stores, pgvector, postgresql, production]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# PGVector

PGVector 是基于 PostgreSQL + pgvector 扩展的**生产级向量存储**，适合已有 PostgreSQL 基础设施的团队。

## 安装

```bash
pip install pgvector langchain-postgres
```

同时需要 PostgreSQL 安装 pgvector 扩展：

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

## 导入

```python
from langchain_postgres import PGVector
from langchain_openai import OpenAIEmbeddings
```

## 基本用法

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."

from langchain_core.documents import Document

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

CONNECTION_STRING = "postgresql+psycopg://user:pass@localhost:5432/vectordb"

vectorstore = PGVector.from_documents(
    documents=[Document(page_content="example content", metadata={"topic": "test"})],
    embedding=embeddings,
    connection=CONNECTION_STRING,
    collection_name="docs",
    pre_delete_collection=False,  # True 则每次重建
)
```

## 相似度搜索

```python
# 基本搜索
results = vectorstore.similarity_search("query", k=4)

# 带元数据过滤
results = vectorstore.similarity_search(
    "query",
    k=5,
    filter={"author": "Alice", "year": {"$gte": 2024}},
)

# 带分数
scored = vectorstore.similarity_search_with_score("query", k=4)
```

## 添加与删除

```python
# 添加文档
vectorstore.add_documents([Document(page_content="new", metadata={})])

# 添加文本
vectorstore.add_texts(["text1", "text2"], metadatas=[{"a": 1}, {"a": 2}])

# 删除
vectorstore.delete(ids=["id1", "id2"][[)
```

## 转为 Retriever

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 4, "filter": {"category": "tech"}},
)
docs = retriever.invoke("search query")
```

## 生产部署建议

- 使用连接池（如 PgBouncer）管理数据库连接
- 为大规模数据建立 HNSW 或 IVFFlat 索引以加速搜索
- `pre_delete_collection=False` 避免每次启动时清空数据
- 定期 vacuum 和 reindex 以保持查询性能

## 注意事项

- 连接字符串格式为 `postgresql+psycopg://...`
- pgvector 扩展必须在数据库中预先安装
- 适合中等规模生产（百万级向量）；超大规模考虑 [[langchain/modules/langchain_vectorstores/Pinecone|Pinecone]] 或 Milvus

## 相关

- [[langchain/modules/langchain_vectorstores/概述|<- 返回 vectorstores 概述]]
