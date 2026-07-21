---
tags: [langchain, vector-stores, in-memory]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# InMemoryVectorStore

`InMemoryVectorStore` 是 LangChain 内置的**内存向量存储**，无需外部依赖，适合开发和测试场景。

## 导入

```python
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings
```

## 基本用法

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 构造
vectorstore = InMemoryVectorStore(embedding=embeddings)

# 添加文本
vectorstore.add_texts(
    texts=["LangChain is a framework", "Python is a language", "AI is transforming tech"],
    metadatas=[{"src": "a"}, {"src": "b"}, {"src": "c"}],
)

# 相似度搜索
results = vectorstore.similarity_search("What is a programming language?", k=2)
for doc in results:
    print(f"- {doc.page_content} | {doc.metadata}")
```

## 带分数搜索

```python
scored = vectorstore.similarity_search_with_score("framework", k=2)
for doc, score in scored:
    print(f"- ({score:.3f}) {doc.page_content}")
```

## 从文档构造

```python
from langchain_core.documents import Document

docs = [
    Document(page_content="LangChain tutorial", metadata={"id": 1}),
    Document(page_content="Python guide", metadata={"id": 2}),
]
vectorstore = InMemoryVectorStore.from_documents(docs, embeddings)
```

## 元数据过滤

```python
results = vectorstore.similarity_search(
    "query",
    k=5,
    filter={"author": "Alice"},
)
```

## 删除文档

```python
ids = ["doc1", "doc2"][[
vectorstore.delete(ids=ids)
```

## 转为 Retriever

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",   # 或 "mmr"（最大边际相关性）
    search_kwargs={"k": 4, "filter": {"category": "tech"}},
)
docs = retriever.invoke("search query")
```

## 注意事项

- **重启即丢失**：所有数据仅存在于内存中
- 不适合生产环境，仅用于开发和测试
- 生产环境请使用 [[langchain/modules/langchain_vectorstores/FAISS|FAISS]]、[[langchain/modules/langchain_vectorstores/Chroma|Chroma]] 或 [[langchain/modules/langchain_vectorstores/PGVector|PGVector]]

## 相关

- [[langchain/modules/langchain_vectorstores/概述|<- 返回 vectorstores 概述]]
