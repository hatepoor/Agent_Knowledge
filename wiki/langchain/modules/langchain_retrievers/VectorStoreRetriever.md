---
tags: [langchain, retrievers, vector-store, similarity, mmr]
type: concept
created: 2026-07-07
updated: 2026-07-13
langchain_version: ">=0.3.0"
---

# VectorStoreRetriever

## 是什么

`VectorStoreRetriever` 是 LangChain 中最基础的检索器——把 VectorStore 包装成统一的 Retriever 接口。**99% 的 RAG 应用从这个类开始。**

通过 `vectorstore.as_retriever()` 一行创建，然后就能塞进任何 LCEL 链。

## 导入

```python
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings
```

---

## 三种检索模式

`search_type` 决定检索算法，**这是最重要的参数**：

| search_type | 原理 | 适用场景 | 性能 |
|-------------|------|----------|------|
| `similarity` | 纯余弦相似度排序，返回 top-k | 事实性问答、精确匹配 | 最快 |
| `mmr` | 在相似度基础上增加多样性惩罚 | 综合性查询、避免返回重复内容 | 较慢（多一轮计算） |
| `similarity_score_threshold` | 只返回相似度超过阈值的文档 | 高精度场景、法律/医疗 | 取决于阈值 |

### similarity（默认）

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 4},
)
results = retriever.invoke("What is LangChain?")
# → 按相似度从高到低返回 4 个文档
```

**什么时候用**：用户问"XX 是什么"这种事实性问题，不需要多样性。

### mmr（最大边际相关性）

```python
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={
        "k": 4,
        "fetch_k": 20,
        "lambda_mult": 0.7,
    },
)
```

**参数解读**：

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `k` | 4 | 最终返回的文档数 |
| `fetch_k` | 20 | 先取回的候选数（> k 才有多样性效果） |
| `lambda_mult` | 0.5 | 0=纯多样性，1=纯相关性 |

**什么时候用**：知识库内容重复度高、需要覆盖多角度的场景。

**`lambda_mult` 调参指南**：

| lambda_mult | 效果 | 适用 |
|-------------|------|------|
| 0.9-1.0 | 几乎等于 similarity | 不需要多样性 |
| 0.6-0.8 | **推荐范围**，平衡相关性和多样性 | 通用场景 |
| 0.3-0.5 | 强多样性 | 知识库内容高度重复 |
| 0.0-0.2 | 可能返回不相关内容 | 不推荐 |

### similarity_score_threshold（相似度阈值）

```python
retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "k": 5,
        "score_threshold": 0.75,
    },
)
```

**什么时候用**：你对结果质量要求极高，宁可返回空也不要给不相关的文档。

**阈值设置建议**：

| score_threshold | 效果 |
|-----------------|------|
| 0.9+ | 极高精度，可能返回 0 条 |
| 0.7-0.8 | **推荐**，平衡精度和召回 |
| 0.5-0.6 | 较宽松 |
| < 0.5 | 几乎不过滤，不推荐 |

> `score_threshold` 不是标准化的——不同 embedding 模型的分值分布差异很大。**用你自己的数据做基准测试**，不要照搬别人的阈值。

---

## 元数据过滤

利用文档的 metadata 字段做结构化过滤：

```python
# 只检索 topic 为 "langchain" 的文档
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={
        "k": 4,
        "filter": {"topic": "langchain"},
    },
)
```

**支持的过滤操作**（取决于 VectorStore 后端）：

```python
# 精确匹配
{"author": "张三"}

# 比较运算
{"year": {"$gte": 2024}}
{"score": {"$lt": 0.5}}

# 列表匹配
{"topic": {"$in": ["python", "langchain"]}}

# 逻辑组合 AND
{"$and": [{"topic": "python"}, {"year": {"$gte": 2024}}]}

# 逻辑组合 OR
{"$or": [{"topic": "python"}, {"topic": "langchain"}]}
```

> FAISS 对复杂过滤支持有限，Chroma 和 PGVector 支持更完整。选择 VectorStore 时把过滤需求考虑进去。

---

## 实战：完整 RAG 链

```python
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_deepseek import ChatDeepSeek

embeddings = OpenAIEmbeddings()
model = ChatDeepSeek(model="deepseek-v4-flash")

# 1. 构建向量库
docs = [
    Document(page_content="LangChain 是一个 LLM 应用开发框架", metadata={"topic": "langchain"}),
    Document(page_content="Python 是一种解释型编程语言", metadata={"topic": "python"}),
    Document(page_content="Rust 注重内存安全和并发性能", metadata={"topic": "rust"}),
]
vectorstore = InMemoryVectorStore.from_documents(docs, embeddings)

# 2. 创建检索器
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 3, "fetch_k": 10, "lambda_mult": 0.7},
)

# 3. 构建 RAG 链
def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

prompt = ChatPromptTemplate.from_template(
    "根据以下上下文回答问题:\n\n{context}\n\n问题: {question}"
)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)

rag_chain.invoke("什么是 LangChain？")
```

---

## 三种模式选择决策树

```
用户问题是：
├── 事实性单一答案（"什么是 XX"） → similarity
├── 需要多角度看（"XX 的优缺点"） → mmr（lambda_mult=0.7）
├── 对精度要求极高    → similarity_score_threshold（score_threshold=0.75）
└── 不确定             → 先用 similarity 跑一遍，看结果质量再决定
```

---

## 故障排查

### 问题 1：MMR 结果和 similarity 没区别

**原因**：`fetch_k` 太小或 `lambda_mult` 太高。

**修正**：`fetch_k` 至少是 `k` 的 3-5 倍，`lambda_mult` 调到 0.5-0.7 区间。

```python
# ❌ 无效的 MMR — fetch_k 太小，没有多样性空间
vectorstore.as_retriever(search_type="mmr", search_kwargs={"k": 4, "fetch_k": 5})

# ✅ fetch_k 至少 3x k
vectorstore.as_retriever(search_type="mmr", search_kwargs={"k": 4, "fetch_k": 15})
```

### 问题 2：阈值检索返回空结果

**原因**：`score_threshold` 设太高，没有文档达到。

**修正**：先跑一次 similarity 看分数分布，再定阈值。

```python
# 诊断：查看实际分数
results_with_scores = vectorstore.similarity_search_with_score("your query", k=10)
for doc, score in results_with_scores:
    print(f"分数: {score:.4f} | {doc.page_content[:50]}...")

# 根据实际分数分布设阈值，通常取中位数
```

### 问题 3：元数据过滤不生效

**原因**：VectorStore 后端不支持该过滤语法，或 metadata 字段名不匹配。

**修正**：检查你的 VectorStore（FAISS 支持有限，PGVector/Chroma 支持完整）。

---

## 子页面

- [[langchain/modules/langchain_retrievers/MultiQueryRetriever|MultiQueryRetriever]] — 查询改写增强召回
- [[langchain/modules/langchain_retrievers/EnsembleRetriever|EnsembleRetriever]] — 多路融合

## 相关

- [[langchain/modules/langchain_retrievers/概述|← 返回 retrievers 概述]]
- [[langchain/modules/langchain_vectorstores/概述|vectorstores 模块]]
