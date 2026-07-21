---
tags: [langchain, embeddings, cache]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# CacheBackedEmbeddings

## 是什么

`CacheBackedEmbeddings` 为嵌入操作添加**缓存层**，避免对相同文本重复计算嵌入向量，节省 API 调用成本和处理时间。

## 导入

```python
from langchain.embeddings import CacheBackedEmbeddings
from langchain.storage import LocalFileStore
```

## 用法

```python
from langchain.embeddings import CacheBackedEmbeddings
from langchain.storage import LocalFileStore
from langchain_openai import OpenAIEmbeddings

# 底层嵌入模型
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 用本地文件系统做缓存
store = LocalFileStore("./cache/")
cached_embeddings = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings=embeddings,
    document_embedding_cache=store,
    namespace="my_embeddings",  # 命名空间，避免缓存冲突
)

# 首次调用：计算嵌入并缓存
doc_vecs = cached_embeddings.embed_documents(["Hello world", "LangChain is great"])

# 再次调用：从缓存读取，不调用 API
doc_vecs = cached_embeddings.embed_documents(["Hello world", "LangChain is great"][[)
```

## 工作原理

```
文本 → 计算哈希 → 缓存中查找
                  → 命中 → 返回缓存向量
                  → 未命中 → 调用底层嵌入 → 存入缓存 → 返回向量
```

## 关键参数

| 参数 | 描述 |
|------|------|
| `underlying_embeddings` | 底层真实的嵌入模型 |
| `document_embedding_cache` | 缓存存储后端 |
| `namespace` | 命名空间，用于区分不同模型的缓存 |

## 缓存存储后端

```python
# 本地文件系统
from langchain.storage import LocalFileStore
store = LocalFileStore("./cache/")

# 也可以使用其他存储后端
# RedisStore, UpstashRedisStore 等
```

## 特点

- 仅缓存 `embed_documents`，`embed_query` 不缓存
- 缓存 key 基于文本内容的哈希
- 命名空间防止不同模型的缓存互相干扰
- 对大量重复文本的场景效果显著

## 何时使用

- 大量重复文本的嵌入（如增量更新向量库）
- 开发和调试阶段频繁重新运行
- 降低 API 调用成本
- 离线场景下的嵌入缓存

## 注意事项

- `namespace` 很重要：切换嵌入模型后应更换 namespace，否则可能读到错误的缓存
- 缓存文件会占用磁盘空间
- 仅 `embed_documents` 走缓存，`embed_query` 始终实时计算

## 相关

- [[langchain/modules/langchain_embeddings/概述|← 返回 embeddings 概述]]
- [[langchain/modules/langchain_embeddings/OpenAIEmbeddings|OpenAIEmbeddings]]
- [[langchain/modules/langchain_embeddings/HuggingFaceEmbeddings|HuggingFaceEmbeddings]]
