---
tags: [langchain, vector-stores, chroma]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# Chroma

Chroma 是一个开源的**向量数据库**，支持本地嵌入式和服务端两种模式，适合快速原型开发和中等规模部署。

## 安装

```bash
pip install chromadb langchain-chroma
```

## 导入

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings
```

## 基本用法

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."

from langchain_core.documents import Document

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 从文档构造（持久化目录）
vectorstore = Chroma.from_documents(
    documents=[Document(page_content="LangChain tutorial", metadata={"id": 1})],
    embedding=embeddings,
    persist_directory="./chroma_db",
    collection_name="my_collection",
)
```

## 相似度搜索与元数据过滤

```python
# 基本搜索
results = vectorstore.similarity_search("LangChain", k=3)

# 带元数据过滤
results = vectorstore.similarity_search(
    "LangChain",
    k=3,
    filter={"id": 1},
)

# 带分数
scored = vectorstore.similarity_search_with_score("LangChain", k=3)
for doc, score in scored:
    print(f"({score:.3f}) {doc.page_content}")
```

> Chroma 的过滤语法使用 `$gte`、`$lte`、`$eq`、`$ne` 等操作符。

## 持久化

Chroma 在指定 `persist_directory` 时自动持久化到磁盘。下次加载时：

```python
vectorstore = Chroma(
    persist_directory="./chroma_db",
    embedding_function=embeddings,
    collection_name="my_collection",
)
```

## 连接远程 Chroma 服务

```python
from chromadb import HttpClient

client = HttpClient(host="localhost", port=8000)
vectorstore = Chroma(
    client=client,
    collection_name="my_collection",
    embedding_function=embeddings,
)
```

## 添加与删除文档

```python
# 添加
vectorstore.add_documents([Document(page_content="new doc", metadata={"tag": "x"})])

# 添加文本
vectorstore.add_texts(["text1", "text2"], metadatas=[{"a": 1}, {"a": 2}])

# 删除
vectorstore.delete(ids=["doc_id_1", "doc_id_2"][[)
```

## 转为 Retriever

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 4, "filter": {"id": 1}},
)
docs = retriever.invoke("search query")
```

## 注意事项

- Chroma 的元数据过滤语法与 Pinecone 不同（`$gte` vs `gte`）
- 嵌入式模式无需额外服务进程
- 服务端模式需要启动 `chroma run` 服务

## 避免重复存储

Chroma 原生支持基于 ID 的 upsert，相同 ID 不会重复插入：

```python
import hashlib

def get_doc_id(doc: Document) -> str:
    """基于内容生成确定性 ID"""
    raw = f"{doc.metadata.get('source', '')}:{doc.page_content[:200]}"
    return hashlib.md5(raw.encode()).hexdigest()

# 给每个块生成 ID
ids = [get_doc_id(chunk) for chunk in chunks]

# Chroma 自动 upsert：相同 ID 更新，新 ID 插入
vectorstore.add_documents(chunks, ids=ids)

# 也可以先检查已有 ID
existing_ids = set(vectorstore.get()["ids"])
new_chunks = [c for c, id in zip(chunks, ids) if id not in existing_ids]
new_ids = [id for id in ids if id not in existing_ids]
if new_chunks:
    vectorstore.add_documents(new_chunks, ids=new_ids)
    print(f"新增 {len(new_chunks)} 个，跳过 {len(chunks) - len(new_chunks)} 个重复")
```

## 相关

- [[langchain/modules/langchain_vectorstores/概述|<- 返回 vectorstores 概述]]
