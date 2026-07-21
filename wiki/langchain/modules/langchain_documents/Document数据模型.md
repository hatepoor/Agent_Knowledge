---
tags: [langchain, documents, data-model]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# Document 数据模型

## 是什么

`Document` 是 LangChain 的**核心文档数据类型**，由 `page_content`（文本内容）和 `metadata`（元数据字典）两部分组成，贯穿整个文档处理流水线。

## 导入

```python
from langchain_core.documents import Document
```

## 结构

```python
class Document:
    page_content: str       # 文档的文本内容
    metadata: dict          # 元数据（来源、作者、页码等）
```

## 基本用法

```python
from langchain_core.documents import Document

doc = Document(
    page_content="LangChain is a framework for LLM applications.",
    metadata={
        "source": "intro.md",
        "author": "Alice",
        "page": 1,
    },
)
print(doc.page_content)  # "LangChain is a framework for LLM applications."
print(doc.metadata)      # {"source": "intro.md", "author": "Alice", "page": 1}
```

## 手工构造多个 Document

```python
docs = [
    Document(
        page_content="First document content",
        metadata={"source": "manual", "id": 1},
    ),
    Document(
        page_content="Second document content",
        metadata={"source": "manual", "id": 2},
    ),
]
```

## 元数据增强

在分割后可以为每个 chunk 添加额外元数据：

```python
def enrich_metadata(doc: Document) -> Document:
    """为每个 chunk 添加额外元数据"""
    doc.metadata["char_count"] = len(doc.page_content)
    doc.metadata["word_count"] = len(doc.page_content.split())
    return doc

enriched = [enrich_metadata(d) for d in split_docs][[
```

## 元数据注意事项

- metadata 会随 Document 在整个 RAG 流水线中传递（分割 → 嵌入 → 向量存储 → 检索）
- 必须使用 JSON 可序列化类型（str、int、float、bool、list）
- 避免嵌套过深或包含大对象
- 常用字段：`source`（来源）、`page`（页码）、`author`（作者）、`date`（日期）

## 何时使用

- 表示从任何来源加载的文档
- RAG 流水线中的基础数据单元
- 向量数据库中的记录
- Retriever 返回的检索结果

## 相关

- [[langchain/modules/langchain_documents/概述|← 返回 documents 概述]]
- [[langchain/modules/langchain_documents/文档加载器|文档加载器]]
- [[langchain/modules/langchain_documents/文本分割器|文本分割器]]
