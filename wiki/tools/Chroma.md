---
tags: [entity, 工具, 向量数据库, 嵌入式数据库]
created: 2026-06-23
updated: 2026-07-07
type: entity
---

# Chroma

## 概述

Chroma 是一个开源的嵌入式向量数据库，专为 AI 和嵌入式应用设计。它是目前最流行的轻量级向量数据库之一，特别适合本地开发和原型验证。

## 核心特性

- **嵌入式**：可以直接嵌入 Python 应用，无需独立服务器
- **简单 API**：极简的 CRUD 接口，快速上手
- **多后端支持**：支持内存存储和持久化存储（SQLite/PostgreSQL）
- **元数据过滤**：支持基于元数据的混合查询
- **多种距离度量**：余弦相似度、欧氏距离、内积

## 在 RAG 中的应用

Chroma 常用于 RAG 系统的向量存储层：
- 存储文档切片的 Embedding 向量
- 基于语义相似度检索相关文档
- 支持元数据过滤（如按来源、日期筛选）

```python
import chromadb
client = chromadb.Client()
collection = client.create_collection("docs")
collection.add(documents=["text1", "text2"], ids=["id1", "id2"])
results = collection.query(query_texts=["query"][[, n_results=2)
```

## 与其他向量数据库对比

| 特性   | Chroma  | Pinecone | Milvus | Weaviate |
| ---- | ------- | -------- | ------ | -------- |
| 部署   | 嵌入式/本地  | 托管云      | 分布式    | 自托管/云    |
| 复杂度  | 低       | 低        | 高      | 中        |
| 扩展性  | 中       | 高        | 很高     | 高        |
| 适用场景 | 原型/中小规模 | 生产/大规模   | 企业级    | 生产级      |

## 详细文档

- [[tools/向量数据库/chroma/概述|Chroma 详细介绍]] — 定位、特性、适用场景
- [[tools/向量数据库/chroma/快速入门|快速入门]] — 安装、基本操作、第一个示例
- [[tools/向量数据库/chroma/核心原理|核心原理]] — HNSW 索引、存储后端、架构设计
- [[tools/向量数据库/chroma/持久化与部署|持久化与部署]] — 数据持久化、Client-Server 模式
- [[tools/向量数据库/chroma/与LangChain集成|与 LangChain 集成]] — RAG 实现、Agent 集成

## 来源

- [[agent/RAG/向量数据库]] — 原文提到 Chroma 作为轻量级向量数据库选项
