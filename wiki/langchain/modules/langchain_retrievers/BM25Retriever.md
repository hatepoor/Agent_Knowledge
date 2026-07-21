---
tags: [langchain, retrievers, bm25, keyword, sparse]
type: concept
created: 2026-07-07
updated: 2026-07-13
langchain_version: ">=0.3.0"
---

# BM25Retriever

## 是什么

`BM25Retriever` 是**基于词频统计的关键词检索器**——不依赖 embedding 模型，不需要 GPU，不需要 API Key。它对精确关键词（代码、ID、术语、产品名）的匹配能力**往往优于向量检索**。

## 安装

```bash
pip install rank_bm25 langchain-community
```

## 导入

```python
# ⚠️ langchain_community 已弃用，新项目建议直接用 rank_bm25 库
from langchain_community.retrievers import BM25Retriever
from langchain_core.documents import Document
```

---

## 基本用法

```python
docs = [
    Document(page_content="LangChain 是一个 LLM 应用开发框架"),
    Document(page_content="Python 在数据科学和 Web 开发中广泛应用"),
    Document(page_content="AI Agent 可以自主调用工具完成复杂任务"),
]

# 从 Document 列表构造
bm25 = BM25Retriever.from_documents(docs)
bm25.k = 3

results = bm25.invoke("LangChain 框架")
# → 返回包含 "LangChain" 和 "框架" 的文档，按 BM25 分数排序
```

### 从纯文本构造

```python
bm25 = BM25Retriever.from_texts(
    ["LangChain 是一个框架", "Python 是一种语言", "AI 正在改变世界"],
)
bm25.k = 2
results = bm25.invoke("Python 语言")
```

---

## BM25 算法入门

BM25 是 TF-IDF 的进化版，核心思想：

1. **词频（TF）**：一个词在文档中出现次数越多，相关性越高（但有上限，防止高频词主导）
2. **逆文档频率（IDF）**：一个词在越少文档中出现，区分度越高（"的"出现在几乎所有文档 → 权重低）
3. **文档长度归一化**：长文档天然包含更多词，需要对抗这种偏差

两个可调参数（通过 `rank_bm25` 传入）：

| 参数 | 默认 | 作用 |
|------|------|------|
| `k1` | 1.5 | 控制 TF 饱和度（0=忽略词频，越大越重视） |
| `b` | 0.75 | 控制文档长度惩罚（0=无惩罚，1=完全惩罚） |

大多数场景**不需要调这两个参数**，默认值已经是最优。

---

## BM25 vs 向量检索

| | BM25 | 向量检索 |
|---|---|---|
| **强项** | 精确关键词（代码、ID、术语、拼音） | 语义相似（同义词、近义词、跨语言） |
| **弱项** | "新能源车" 搜不到 "电动汽车" | 搜 "API_KEY_ENV" 可能返回无关的环境变量文档 |
| **成本** | 免费，CPU 即可 | 需要 embedding API 调用或 GPU |
| **速度** | ms 级 | 取决于 embedding 模型 |
| **更新** | 增删文档后重建索引（快） | 增删向量（取决于 VectorStore） |

> **互补关系，不是替代关系。** 生产环境的最佳实践是 **BM25 + 向量检索组合使用**。

---

## 与 EnsembleRetriever 配合（标准用法）

```python
from langchain.retrievers import EnsembleRetriever
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings

# 路1: 关键词
bm25 = BM25Retriever.from_documents(docs)
bm25.k = 5

# 路2: 语义
vectorstore = InMemoryVectorStore.from_documents(docs, OpenAIEmbeddings())
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# 融合（推荐权重）
ensemble = EnsembleRetriever(
    retrievers=[bm25, vector_retriever],
    weights=[0.4, 0.6],  # 语义略重要
)

results = ensemble.invoke("怎么使用 LangChain agent？")
```

---

## 什么时候该用 / 不该用

| 场景 | 推荐 |
|------|------|
| 代码搜索（搜函数名、变量名） | ✅ BM25 优先 |
| 产品名、ID、编号搜索 | ✅ BM25 优先 |
| 中文自然语言问答 | ⚠️ BM25 不够，必须加向量 |
| 跨语言搜索 | ❌ BM25 不行，用向量 |
| 同义词、近义词匹配 | ❌ BM25 不行，用向量 |
| 快速原型 | ✅ BM25 0 成本启动 |
| 离线/无 API 环境 | ✅ BM25 完全本地 |

---

## 注意事项

- **BM25 是统计模型，不"理解"语义**——"车"和"汽车"在 BM25 眼里是完全不同的词
- **支持中文分词**——`rank_bm25` 默认按空格分词，中文需要用 jieba 等分词器预处理
- **不支持元数据过滤**——只能按文本内容检索，不能 filter metadata
- **文档集变更后需要重建**——`from_documents` 会重新计算 IDF，不是增量更新

---

## 相关

- [[langchain/modules/langchain_retrievers/概述|← 返回 retrievers 概述]]
- [[langchain/modules/langchain_retrievers/EnsembleRetriever|EnsembleRetriever]] — BM25 最常见的搭档
