---
tags: [langchain, vector-stores, pinecone, cloud]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# Pinecone

Pinecone 是一个**全托管云向量数据库**服务，适合大规模生产部署，无需管理基础设施。

## 安装

```bash
pip install pinecone-client langchain-pinecone
```

## 导入

```python
from langchain_pinecone import PineconeVectorStore
from pinecone import Pinecone
from langchain_openai import OpenAIEmbeddings
```

## 基本用法

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 初始化 Pinecone 客户端
pc = Pinecone(api_key="your-pinecone-api-key")

# 连接到已有索引
vectorstore = PineconeVectorStore(
    index=pc.Index("my-index"),
    embedding=embeddings,
    text_key="text",
)
```

> 使用前需在 [Pinecone 控制台](https://app.pinecone.io) 创建索引，维度需与 embedding 模型匹配。

## 相似度搜索

```python
# 基本搜索
results = vectorstore.similarity_search("query", k=4)

# 带元数据过滤（Pinecone 语法）
results = vectorstore.similarity_search(
    "query",
    k=5,
    filter={"author": {"$eq": "Alice"}, "year": {"$gte": 2024}},
)

# 带分数
scored = vectorstore.similarity_search_with_score("query", k=4)
```

> Pinecone 过滤操作符使用 `$eq`、`$gte`、`$in` 等，与 Chroma 的语法略有不同。

## 添加文档

```python
from langchain_core.documents import Document

docs = [
    Document(page_content="doc1 content", metadata={"author": "Alice"}),
    Document(page_content="doc2 content", metadata={"author": "Bob"}),
]
ids = vectorstore.add_documents(docs)
```

## 删除文档

```python
vectorstore.delete(ids=["id1", "id2"][[)

# 删除所有文档（谨慎操作）
vectorstore.delete(delete_all=True)
```

## 转为 Retriever

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 4, "filter": {"category": "tech"}},
)
docs = retriever.invoke("search query")
```

## 命名空间

Pinecone 支持命名空间（namespace）来隔离数据：

```python
vectorstore = PineconeVectorStore(
    index=pc.Index("my-index"),
    embedding=embeddings,
    namespace="production-data",
)
```

## 注意事项

- 索引维度必须与 embedding 模型维度匹配（如 `text-embedding-3-small` 为 1536 维）
- Pinecone 过滤语法与其他后端不同，注意操作符前缀 `$`
- 免费套餐有向量数量限制，生产环境需付费计划
- 适合大规模（十亿级向量）生产部署

## 相关

- [[langchain/modules/langchain_vectorstores/概述|<- 返回 vectorstores 概述]]
