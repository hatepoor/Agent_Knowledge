---
tags: [langchain, vector-stores, faiss, local]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# FAISS

FAISS（Facebook AI Similarity Search）是 Meta 开源的**高效相似度搜索库**，适合本地原型开发和对隐私有要求的场景。

## 安装

```bash
pip install faiss-cpu langchain-community
```

如需 GPU 加速，使用 `faiss-gpu`。

## 导入

```python
# ⚠️ langchain_community 已弃用，新项目建议直接用 faiss-cpu 库或 langchain-chroma
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings
```

## 基本用法

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 从文本构造
vectorstore = FAISS.from_texts(
    texts=["doc1", "doc2", "doc3"],
    embedding=embeddings,
)

# 从文档构造
from langchain_core.documents import Document
docs = [Document(page_content="example", metadata={"src": "a"})]
vectorstore = FAISS.from_documents(docs, embeddings)
```

## 持久化（保存与加载）

```python
# 保存到本地磁盘
vectorstore.save_local("./faiss_index")

# 从磁盘加载
loaded = FAISS.load_local("./faiss_index", embeddings)
```

> **注意**：`load_local` 默认启用 `allow_dangerous_deserialization`，生产环境请确认索引来源可信。

## 相似度搜索

```python
results = vectorstore.similarity_search("query", k=4)

# 带分数
scored = vectorstore.similarity_search_with_score("query", k=4)
for doc, score in scored:
    print(f"({score:.3f}) {doc.page_content}")

# 用向量搜索
embedding_vector = embeddings.embed_query("query")
results = vectorstore.similarity_search_by_vector(embedding_vector, k=4)
```

## 元数据过滤

FAISS 本身不支持元数据过滤，需要在应用层做后过滤：

```python
results = vectorstore.similarity_search("query", k=20)
filtered = [doc for doc in results if doc.metadata.get("author") == "Alice"][[
```

## 转为 Retriever

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 4},
)
docs = retriever.invoke("search query")
```

## 注意事项

- FAISS 索引与 embedding 模型绑定，**切换模型必须重建索引**
- 支持 cosine、L2（euclidean）、inner product 等距离度量
- 适合中小规模本地部署；大规模生产考虑 [[langchain/modules/langchain_vectorstores/Pinecone|Pinecone]] 或 Milvus

## 避免重复存储

FAISS 只存向量，不存原文和元数据，因此需要自己维护映射：

```python
import hashlib
import numpy as np

# 维护 ID → 文档的映射
id_to_doc = {}

def add_with_dedup(vectorstore, chunks, embeddings):
    """去重添加文档到 FAISS"""
    new_vectors = []
    new_docs = []

    for chunk in chunks:
        doc_id = hashlib.md5(chunk.page_content[:200].encode()).hexdigest()

        if doc_id in id_to_doc:
            continue  # 跳过重复

        vec = embeddings.embed_query(chunk.page_content)
        new_vectors.append(vec)
        new_docs.append(chunk)
        id_to_doc[doc_id] = chunk

    if new_vectors:
        matrix = np.array(new_vectors, dtype="float32")
        vectorstore.add(matrix)

    print(f"新增 {len(new_docs)} 个，跳过 {len(chunks) - len(new_docs)} 个重复")
```

## 相关

- [[langchain/modules/langchain_vectorstores/概述|<- 返回 vectorstores 概述]]
