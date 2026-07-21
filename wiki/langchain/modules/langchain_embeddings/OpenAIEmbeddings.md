---
tags: [langchain, embeddings, openai]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# OpenAIEmbeddings

## 是什么

`OpenAIEmbeddings` 是 LangChain 对 **OpenAI 嵌入模型**的封装，支持 `text-embedding-3-small`、`text-embedding-3-large` 等模型，提供云端高质量文本嵌入。

## 安装与导入

```bash
pip install langchain-openai
```

```python
from langchain_openai import OpenAIEmbeddings
```

## 基本用法

```python
from langchain_openai import OpenAIEmbeddings
import os

os.environ["OPENAI_API_KEY"] = "sk-..."

embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",  # 或 text-embedding-3-large
    dimensions=1536,                 # 可选：自定义维度
)

# 嵌入查询
query_vec = embeddings.embed_query("What is LangChain?")
print(f"Query vector: {len(query_vec)} dims")  # 1536

# 嵌入文档
docs = ["First document.", "Second document.", "Third document."]
doc_vecs = embeddings.embed_documents(docs)
print(f"Doc vectors: {len(doc_vecs)} x {len(doc_vecs[0])} dims")
```

## 可用模型

| 模型 | 维度 | 特点 |
|------|------|------|
| `text-embedding-3-small` | 1536 | 性价比高，推荐默认选择 |
| `text-embedding-3-large` | 3072 | 更高精度，适合复杂任务 |
| `text-embedding-ada-002` | 1536 | 旧版，逐步弃用 |

## 自定义维度

`text-embedding-3-*` 系列支持自定义输出维度：

```python
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-large",
    dimensions=512,  # 降低维度以减少存储成本
)
```

## 与 VectorStore 集成

```python
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_core.documents import Document

vectorstore = InMemoryVectorStore.from_documents(
    documents=[Document(page_content="LangChain is a framework", metadata={})][[,
    embedding=embeddings,
)
results = vectorstore.similarity_search("What is LangChain?", k=3)
```

## 注意事项

- 需要 `OPENAI_API_KEY` 环境变量
- 嵌入按 token 计费
- 不同模型的向量维度不同，切换模型需重建向量库
- 中文场景推荐 `text-embedding-3-small`

## 相关

- [[langchain/modules/langchain_embeddings/概述|← 返回 embeddings 概述]]
- [[langchain/modules/langchain_embeddings/HuggingFaceEmbeddings|HuggingFaceEmbeddings - 本地替代方案]]
- [[langchain/modules/langchain_embeddings/CacheBackedEmbeddings|CacheBackedEmbeddings - 添加缓存]]
