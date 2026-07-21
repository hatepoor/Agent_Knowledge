---
tags: [langchain, retrievers, self-query, metadata, nlp-filtering]
type: concept
created: 2026-07-07
updated: 2026-07-13
langchain_version: ">=0.3.0"
---

# SelfQueryRetriever

## 是什么

用户说"找 2024 年关于 Python 的文章"，你不想让用户手动构造 `filter={"topic": "python", "year": 2024}`。`SelfQueryRetriever` 让 LLM 从自然语言中**自动解析过滤条件**，拆成"搜什么 + 怎么过滤"。

```
用户: "找 2024 年关于 Python 并且评分大于 4 的文章"
    ↓ LLM 解析
query: "Python 文章"
filter: {"topic": "python", "year": 2024, "rating": {"$gt": 4}}
    ↓ 向量搜索 + 元数据过滤
返回符合条件的结果
```

## 导入

```python
from langchain.retrievers import SelfQueryRetriever
from langchain.chains.query_constructor.schema import AttributeInfo
from langchain_openai import ChatOpenAI
```

---

## 基本用法

```python
from langchain_core.documents import Document
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings, ChatOpenAI

embeddings = OpenAIEmbeddings()
llm = ChatOpenAI(model="gpt-4o", temperature=0)

docs = [
    Document(page_content="LangChain 是一个 LLM 框架", metadata={"topic": "langchain", "year": 2024, "rating": 4.5}),
    Document(page_content="Python 在数据科学中广泛应用", metadata={"topic": "python", "year": 2023, "rating": 4.8}),
    Document(page_content="AI Agent 可以自主使用工具", metadata={"topic": "ai", "year": 2025, "rating": 4.2}),
    Document(page_content="LangGraph 是图编排引擎", metadata={"topic": "langgraph", "year": 2024, "rating": 3.9}),
]
vectorstore = InMemoryVectorStore.from_documents(docs, embeddings)

# 1. 定义元数据字段（告诉 LLM 有哪些字段可以过滤）
metadata_field_info = [
    AttributeInfo(name="topic", description="文档的主题", type="string"),
    AttributeInfo(name="year", description="发布年份", type="integer"),
    AttributeInfo(name="rating", description="评分（1-5）", type="float"),
]

# 2. 文档内容描述（可选，帮 LLM 理解内容类型）
document_content_description = "技术文档，涵盖编程框架和 AI 主题"

# 3. 创建 SelfQueryRetriever
self_retriever = SelfQueryRetriever.from_llm(
    llm=llm,
    vectorstore=vectorstore,
    document_content_description=document_content_description,
    metadata_field_info=metadata_field_info,
    verbose=True,  # 开发时打开，看 LLM 解析出了什么
)

# 自然语言查询
results = self_retriever.invoke("找 2024 年的 LangChain 文章")
# 内部解析: query="LangChain 文章", filter={"topic": "langchain", "year": 2024}
```

---

## 支持的过滤操作

LLM 能自动生成以下过滤表达式：

```python
# 精确匹配
"关于 Python 的文档"  →  {"topic": "python"}

# 数值比较
"2024 年以后的文章"    →  {"year": {"$gte": 2024}}
"评分 4 分以上"      →  {"rating": {"$gt": 4}}

# 列表匹配
"Python 或 LangChain 的文章" → {"topic": {"$in": ["python", "langchain"]}}

# 逻辑组合 AND
"2024 年关于 Python 的" → {"$and": [{"topic": "python"}, {"year": 2024}]}

# 逻辑组合 OR
"LangChain 或 AI Agent 的内容" → {"$or": [{"topic": "langchain"}, {"topic": "ai"}]}

# 复杂组合
"2024 年评分 4 分以上的 LangChain 或 Python 文章"
→ {"$and": [
    {"topic": {"$in": ["langchain", "python"]}},
    {"year": 2024},
    {"rating": {"$gt": 4}}
]}
```

---

## AttributeInfo 详细说明

| 字段 | 说明 | 示例 |
|------|------|------|
| `name` | 元数据字段名（必须和 Document.metadata 的 key 一致） | `"year"` |
| `description` | 字段的中文描述（LLM 据此理解字段含义） | `"发布年份"` |
| `type` | `"string"` / `"integer"` / `"float"` / `"boolean"` | `"integer"` |

```python
metadata_field_info = [
    AttributeInfo(
        name="category",
        description="文章分类：技术教程、产品文档、行业分析",
        type="string",
    ),
    AttributeInfo(
        name="word_count",
        description="文章字数",
        type="integer",
    ),
    AttributeInfo(
        name="is_published",
        description="是否已发布",
        type="boolean",
    ),
]
```

---

## VectorStore 兼容性

**不是所有 VectorStore 都支持复杂过滤。** 不同后端的差距很大：

| VectorStore | 等值过滤 | 比较运算 | $in | $and/$or |
|-------------|:---:|:---:|:---:|:---:|
| Chroma | ✅ | ✅ | ✅ | ✅ |
| PGVector | ✅ | ✅ | ✅ | ✅ |
| FAISS | ✅ | ❌ | ❌ | ❌ |
| Qdrant | ✅ | ✅ | ✅ | ✅ |
| InMemoryVectorStore | 有限 | 有限 | 有限 | 有限 |

> FAISS 和 InMemoryVectorStore 只适合原型验证。生产环境用 Chroma/PGVector/Qdrant 这类支持完整过滤语义的后端。

---

## 故障排查

### 问题 1：LLM 解析的 filter 不正确

**原因**：`AttributeInfo` 的 `description` 不够清晰，或者字段名和 metadata 中的 key 不匹配。

**修正**：

```python
# ❌ description 太抽象
AttributeInfo(name="cat", description="类别", type="string")

# ✅ 把可能的取值写进 description
AttributeInfo(name="category", description="文章分类，可选值: tech/policy/market", type="string")

# 开启 verbose 看 LLM 实际返回了什么
self_retriever = SelfQueryRetriever.from_llm(..., verbose=True)
```

### 问题 2：过滤器生效但结果为空

**原因**：LLM 解析过于严格，或者 VectorStore 不支持该过滤操作。

**修正**：
1. 先用 `verbose=True` 查看实际生成的 filter
2. 检查 VectorStore 是否支持所用的过滤操作
3. 如果 filter 正确但结果为空，放宽条件或换用支持更好的 VectorStore

### 问题 3：字段 type 不匹配

```python
# ❌ 如果 metadata 中存的是 int，但 AttributeInfo 声明为 string
AttributeInfo(name="year", type="string")   # LLM 会生成 {"year": "2024"}
# → 实际 metadata 是 {"year": 2024}，过滤失败

# ✅ 类型必须匹配
AttributeInfo(name="year", type="integer")
```

---

## 子页面

- [[langchain/modules/langchain_retrievers/VectorStoreRetriever|VectorStoreRetriever]] — 底层检索器
- [[langchain/modules/langchain_vectorstores/概述|vectorstores 模块]] — 不同后端的过滤能力对比

## 相关

- [[langchain/modules/langchain_retrievers/概述|← 返回 retrievers 概述]]
