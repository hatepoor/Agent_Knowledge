---
tags: [langchain, retrievers, ensemble, rrf, fusion, multi-path]
type: concept
created: 2026-07-07
updated: 2026-07-13
langchain_version: ">=0.3.0"
---

# EnsembleRetriever

## 是什么

同一句话，"向量搜"和"关键词搜"命中的文档可能完全不同。`EnsembleRetriever` 把**多个不同检索器**的结果通过 RRF（Reciprocal Rank Fusion）算法融合，互补各自的盲区。

```
query: "LangChain agent 怎么用？"
    ├── BM25Retriever  →  [Doc3, Doc1, Doc5]  关键词匹配优先
    ├── VectorRetriever → [Doc2, Doc1, Doc4]  语义相似优先
    └── RRF 融合
         → [Doc1, Doc2, Doc3, Doc4, Doc5]  综合排名
```

## RRF 算法

```
RRF_score(d) = Σ [ 1 / (k + rank_i(d)) ]
```

- `rank_i(d)`：文档 d 在第 i 个检索器中的排名（从 1 开始）
- `k`：常数（默认 60），防止单路排名过高
- 所有路结果加权求和，按总分重新排序

```python
# 例子：文档 A 在 BM25 排第 1，在向量排第 3
RRF(A) = 1/(60+1) + 1/(60+3) = 0.0164 + 0.0159 = 0.0323

# 文档 B 在 BM25 排第 2，在向量排第 2
RRF(B) = 1/(60+2) + 1/(60+2) = 0.0161 + 0.0161 = 0.0322

# A > B，虽然 B 更稳定，但 A 有一路排第一
```

## 导入

```python
from langchain.retrievers import EnsembleRetriever
# ⚠️ langchain_community 已弃用
from langchain_community.retrievers import BM25Retriever
```

---

## 基本用法

```python
from langchain_core.documents import Document
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

docs = [
    Document(page_content="LangChain 是一个 LLM 应用开发框架", metadata={"topic": "langchain"}),
    Document(page_content="Python 在数据科学中广泛应用", metadata={"topic": "python"}),
    Document(page_content="AI Agent 可以自主使用工具", metadata={"topic": "ai"}),
    Document(page_content="LangGraph 是 LangChain 的图编排引擎", metadata={"topic": "langgraph"}),
]

# 检索器 1: BM25 关键词
bm25 = BM25Retriever.from_documents(docs)
bm25.k = 3

# 检索器 2: 向量语义
vectorstore = InMemoryVectorStore.from_documents(docs, embeddings)
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# 融合
ensemble = EnsembleRetriever(
    retrievers=[bm25, vector_retriever],
    weights=[0.5, 0.5],
)

results = ensemble.invoke("LangChain framework")
# 同时利用了关键词 "LangChain" 和语义 "framework ≈ 开发框架"
```

---

## 权重调参策略

| 场景 | BM25 权重 | 向量权重 | 理由 |
|------|:-------:|:-------:|------|
| 精确匹配优先（代码、ID、术语） | 0.7 | 0.3 | BM25 对精确词匹配更强 |
| 语义理解优先（自然语言问题） | 0.3 | 0.7 | 向量对同义词、近义词更好 |
| 均衡 | 0.5 | 0.5 | 不确定时从这里开始 |
| 三路以上 | 按路平均 | 按路平均 | 看数据特征分配 |

```python
# 精确匹配优先：搜代码片段、API 名称
ensemble = EnsembleRetriever(
    retrievers=[bm25, vector_retriever],
    weights=[0.7, 0.3],
)

# 语义优先：搜自然语言文档
ensemble = EnsembleRetriever(
    retrievers=[bm25, vector_retriever],
    weights=[0.3, 0.7],
)
```

---

## 三路融合：BM25 + 稠密向量 + 稀疏向量

```python
# 三路互补：关键词 + 语义 + 稀疏语义
ensemble = EnsembleRetriever(
    retrievers=[
        bm25_retriever,        # 关键词
        dense_retriever,       # 稠密向量（embedding）
        sparse_retriever,      # 稀疏向量（如 SPLADE）
    ],
    weights=[0.3, 0.5, 0.2],
    c=60,  # RRF 常数
)
```

---

## EnsembleRetriever vs MultiQueryRetriever

| | EnsembleRetriever | MultiQueryRetriever |
|---|---|---|
| **解决问题** | 检索器覆盖面不够 | query 表述不好 |
| **原理** | 同一 query，多个检索器 | 同一检索器，多个 query |
| **LLM 调用** | 0 | 1 次（生成变体） |
| **适合** | 异构检索器融合 | 用户问题模糊 |
| **组合使用** | ✅ 可以套在 MultiQuery 里面 | ✅ 可以对 Ensemble 的每个 query 做多路融合 |

```python
# 高级组合：先用 MultiQuery 改写，再对每个变体用 Ensemble 检索
base_ensemble = EnsembleRetriever(
    retrievers=[bm25, vector_retriever],
    weights=[0.5, 0.5],
)
# 但这会大幅增加成本，每个变体都触发多路检索
# 不推荐在生产环境这样玩
```

---

## 故障排查

### 问题 1：融合后结果比单路还差

**原因**：给某个表现差的检索器太高权重。

**修正**：分别在测试集上跑 A/B 测试，看每个检索器独立的 recall@k，再分配权重。

```python
# 诊断：单独测试每路的召回
def recall_at_k(retriever, query, ground_truth, k=5):
    results = retriever.invoke(query)
    retrieved_ids = {doc.metadata.get("id") for doc in results[:k]}
    hits = len(retrieved_ids & set(ground_truth))
    return hits / len(ground_truth)

# 根据每路的 recall 结果分配权重
bm25_recall = recall_at_k(bm25, ...)      # 假设 0.6
vector_recall = recall_at_k(vector, ...)   # 假设 0.8
# weight ∝ recall
```

### 问题 2：结果重复

**原因**：多路返回了相同文档，EnsembleRetriever **不去重**——RRF 只会对不同文档排名，相同文档会合并分数。

**实际效果**：相同文档的 RRF 分数会是多路分数之和，排名会自动靠前，但最终结果中**不会出现重复**。

### 问题 3：某一路 retriever 不返回结果

**原因**：BM25 需要文档内容，向量检索器需要 embedding。如果某一路没准备好（空文档集），它会返回空列表。

**修正**：检查每个子检索器在初始化时是否已经加载了文档。

---

## 相关

- [[langchain/modules/langchain_retrievers/概述|← 返回 retrievers 概述]]
- [[langchain/modules/langchain_retrievers/BM25Retriever|BM25Retriever]] — 最常见的 Ensemble 搭档
- [[langchain/modules/langchain_retrievers/MultiQueryRetriever|MultiQueryRetriever]] — 另一种提升召回率的路径
