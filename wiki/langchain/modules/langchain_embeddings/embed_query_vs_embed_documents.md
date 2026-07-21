---
tags: [langchain, embeddings, api]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# embed_query vs embed_documents

## 是什么

LangChain 的 `Embeddings` 抽象类定义了两个核心方法：`embed_query` 和 `embed_documents`，分别用于嵌入**单个查询**和**多个文档**。

## 两个方法

```python
from langchain_core.embeddings import Embeddings

class Embeddings:
    def embed_documents(self, texts: list[str][[) -> list[list[float]]:
        """嵌入多个文档文本，返回向量列表"""
        ...

    def embed_query(self, text: str) -> list[float]:
        """嵌入单个查询文本，返回单个向量"""
        ...
```

## embed_documents - 嵌入多个文档

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

docs = ["First document.", "Second document.", "Third document."]
doc_vecs = embeddings.embed_documents(docs)
print(f"Doc vectors: {len(doc_vecs)} x {len(doc_vecs[0])} dims")
# Doc vectors: 3 x 1536 dims
```

- 用于**构建向量库**时批量嵌入文档
- 输入：`list[str][[`（多个文本）
- 输出：`list[list[float]]`（多个向量）

## embed_query - 嵌入单个查询

```python
query_vec = embeddings.embed_query("What is LangChain?")
print(f"Query vector: {len(query_vec)} dims")
# Query vector: 1536 dims
```

- 用于**检索时**嵌入用户查询
- 输入：`str`（单个文本）
- 输出：`list[float][[`（单个向量）

## 为什么要分离？

| 方面 | embed_documents | embed_query |
|------|----------------|-------------|
| 输入数量 | 多个 | 单个 |
| 典型场景 | 构建向量库 | 检索查询 |
| 调用频率 | 一次性/批量 | 每次查询 |
| 某些模型处理 | 标准嵌入 | 可能有特殊处理 |

- 某些 embedding 模型（如 OpenAI）对 query 和 document 有不同的处理策略
- 分离入口提供了灵活性，允许为两者使用不同的模型
- 批量处理 vs 单条处理的性能优化不同

## 相关

- [[langchain/modules/langchain_embeddings/概述|← 返回 embeddings 概述]]
- [[langchain/modules/langchain_embeddings/OpenAIEmbeddings|OpenAIEmbeddings]]
- [[langchain/modules/langchain_embeddings/HuggingFaceEmbeddings|HuggingFaceEmbeddings]]
