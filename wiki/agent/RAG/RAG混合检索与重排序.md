---
tags: [concept, RAG, 混合检索, 重排序, BM25, reranking]
type: concept
created: 2026-07-08
updated: 2026-07-08
---

# RAG 混合检索与重排序

> 📌 **历史遗留页面**：权威内容见 [[rag/06_混合检索/混合检索]]、[[rag/08_交叉编码器重排/重排序与交叉编码器]]。对照见 [[agent/RAG/README|RAG 迁移对照表]]。

> 单一检索方式总有局限：向量检索擅长语义理解但可能遗漏精确匹配，BM25 擅长关键词匹配但无法理解语义。混合检索结合两者优势，重排序进一步提升精度。

## 检索方式对比

| 方式 | 原理 | 优势 | 劣势 | 适用场景 |
|------|------|------|------|---------|
| **向量检索** | 语义相似度（余弦距离） | 理解同义词、语义关联 | 精确匹配弱（"RAG" 搜不到 "检索增强生成"） | 语义搜索、概念查询 |
| **BM25** | 词频-逆文档频率 | 精确匹配强、可解释性好 | 无法理解语义 | 关键词搜索、术语查询 |
| **混合检索** | 两者结合 | 兼顾语义和精确匹配 | 实现复杂、需要分数融合 | 生产环境、高精度要求 |

## 混合检索实现

### 方法一：Ensemble Retriever（LangChain）

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_chroma import Chroma

# 1. 创建 BM25 检索器
bm25_retriever = BM25Retriever.from_documents(
    documents,
    k=10,
)

# 2. 创建向量检索器
vectorstore = Chroma.from_documents(documents, embeddings)
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})

# 3. 组合为 Ensemble Retriever
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.4, 0.6],  # BM25 权重 0.4，向量权重 0.6
)

# 使用
results = ensemble_retriever.invoke("什么是 RAG？")
```

**原理**：对两种检索结果加权融合，按综合分数排序。

### 方法二：RRF（Reciprocal Rank Fusion）

RRF 是最常用的分数融合算法，不依赖原始分数，只依赖排名。

```python
def reciprocal_rank_fusion(search_results_list, k=60):
    """
    RRF 分数融合
    
    公式：RRF_score(d) = Σ 1 / (k + rank_i(d))
    其中 k 是常数（通常 60），rank_i(d) 是文档 d 在第 i 个检索结果中的排名
    """
    fused_scores = {}
    
    for results in search_results_list:
        for rank, doc in enumerate(results):
            doc_id = doc.metadata.get("id", doc.page_content[:50])
            if doc_id not in fused_scores:
                fused_scores[doc_id] = {"doc": doc, "score": 0}
            fused_scores[doc_id]["score"] += 1 / (k + rank + 1)
    
    # 按融合分数排序
    reranked = sorted(
        fused_scores.values(),
        key=lambda x: x["score"],
        reverse=True
    )
    
    return [item["doc"] for item in reranked]

# 使用
bm25_results = bm25_retriever.invoke("什么是 RAG？")
vector_results = vector_retriever.invoke("什么是 RAG？")

fused_results = reciprocal_rank_fusion([bm25_results, vector_results])
```

**优点**：
- 不依赖原始分数（不同检索器的分数不可比）
- 简单高效
- 对异常值鲁棒

### 方法三：加权线性组合

```python
def weighted_score_fusion(bm25_results, vector_results, alpha=0.5):
    """
    加权线性组合
    
    final_score = alpha * bm25_score + (1-alpha) * vector_score
    """
    doc_scores = {}
    
    # BM25 分数（归一化到 0-1）
    max_bm25 = max(r.metadata.get("score", 1) for r in bm25_results)
    for doc in bm25_results:
        doc_id = doc.metadata.get("id")
        score = doc.metadata.get("score", 0) / max_bm25
        doc_scores[doc_id] = {"doc": doc, "bm25": score, "vector": 0}
    
    # 向量分数（归一化到 0-1）
    for doc in vector_results:
        doc_id = doc.metadata.get("id")
        score = 1 - doc.metadata.get("distance", 0)  # 距离转相似度
        if doc_id in doc_scores:
            doc_scores[doc_id]["vector"] = score
        else:
            doc_scores[doc_id] = {"doc": doc, "bm25": 0, "vector": score}
    
    # 融合
    for doc_id, scores in doc_scores.items():
        scores["final"] = alpha * scores["bm25"] + (1 - alpha) * scores["vector"]
    
    # 排序
    reranked = sorted(
        doc_scores.values(),
        key=lambda x: x["final"],
        reverse=True
    )
    
    return [item["doc"] for item in reranked]
```

## 重排序（Reranking）

重排序是在初步检索后，用更精确的模型对结果重新排序。

### 为什么需要重排序？

```
初步检索（Bi-Encoder）：
- 速度快（向量相似度计算）
- 精度一般（双塔模型，查询和文档独立编码）

重排序（Cross-Encoder）：
- 速度慢（需要逐个计算查询-文档对）
- 精度高（联合编码查询和文档，捕捉交互特征）
```

### Cross-Encoder 原理

```python
from sentence_transformers import CrossEncoder

# 加载模型
model = CrossEncoder("BAAI/bge-reranker-v2-m3")

# 计算查询-文档相关性分数
query = "什么是 RAG？"
documents = [
    "RAG 是检索增强生成技术",
    "Python 是一种编程语言",
    "RAG 通过检索外部知识来增强 LLM",
]

# Cross-Encoder 同时编码查询和文档
pairs = [(query, doc) for doc in documents]
scores = model.predict(pairs)

# 按分数排序
ranked_indices = scores.argsort()[::-1]
for idx in ranked_indices:
    print(f"{scores[idx]:.4f}: {documents[idx]}")
# 0.9523: RAG 是检索增强生成技术
# 0.8912: RAG 通过检索外部知识来增强 LLM
# 0.1234: Python 是一种编程语言
```

### LangChain 重排序实现

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

# 1. 创建基础检索器
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

# 2. 创建重排序器
model = HuggingFaceCrossEncoder(model_name="BAAI/bge-reranker-v2-m3")
compressor = CrossEncoderReranker(model=model, top_n=5)

# 3. 组合为压缩检索器
reranking_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever,
)

# 使用：先检索 20 个，重排序后返回 Top-5
results = reranking_retriever.invoke("什么是 RAG？")
```

### 主流重排序模型

| 模型 | 类型 | 维度 | 中文能力 | 速度 | 适用场景 |
|------|------|------|----------|------|---------|
| **bge-reranker-v2-m3** | 开源 | - | 极强 | 中 | 中文生产环境（推荐） |
| **bge-reranker-large** | 开源 | - | 强 | 慢 | 高精度要求 |
| **Cohere rerank-v3** | 商业 API | - | 强 | 快 | 英文为主，多语言 |
| **Jina reranker-v2** | 商业 API | - | 中 | 快 | 长文档（8K tokens） |
| **ms-marco-MiniLM** | 开源 | - | 弱 | 极快 | 轻量级、CPU 可用 |

### 重排序参数调优

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `top_n` | 重排序后保留的文档数 | 3-5（与最终需要的上下文数一致） |
| `base_retriever.k` | 初步检索的文档数 | 10-20（重排序的候选池） |
| `score_threshold` | 最低相关性阈值 | 0.3-0.5（过滤低质量结果） |

## 完整混合检索 + 重排序流程

```python
from langchain.retrievers import EnsembleRetriever, ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.retrievers import BM25Retriever
from langchain_community.cross_encoders import HuggingFaceCrossEncoder
from langchain_chroma import Chroma

def create_advanced_retriever(documents, embeddings):
    """创建混合检索 + 重排序的完整检索器"""
    
    # 1. BM25 检索器
    bm25_retriever = BM25Retriever.from_documents(documents, k=15)
    
    # 2. 向量检索器
    vectorstore = Chroma.from_documents(documents, embeddings)
    vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 15})
    
    # 3. 混合检索（Ensemble）
    ensemble_retriever = EnsembleRetriever(
        retrievers=[bm25_retriever, vector_retriever],
        weights=[0.4, 0.6][[,
    )
    
    # 4. 重排序
    reranker_model = HuggingFaceCrossEncoder(
        model_name="BAAI/bge-reranker-v2-m3"
    )
    reranker = CrossEncoderReranker(
        model=reranker_model,
        top_n=5,
    )
    
    # 5. 组合
    advanced_retriever = ContextualCompressionRetriever(
        base_compressor=reranker,
        base_retriever=ensemble_retriever,
    )
    
    return advanced_retriever

# 使用
retriever = create_advanced_retriever(documents, embeddings)
results = retriever.invoke("LangChain 的 Agent 如何工作？")
# 流程：BM25 检索 15 个 + 向量检索 15 个 → 融合去重 → 重排序 → 返回 Top-5
```

## 性能优化

### 缓存策略

```python
from functools import lru_cache

class CachedRetriever:
    """带缓存的检索器"""
    
    def __init__(self, retriever, cache_size=1000):
        self.retriever = retriever
        self._retrieve = lru_cache(maxsize=cache_size)(self._retrieve_impl)
    
    def _retrieve_impl(self, query):
        return self.retriever.invoke(query)
    
    def invoke(self, query):
        return self._retrieve(query)
    
    def clear_cache(self):
        self._retrieve.cache_clear()
```

### 异步重排序

```python
import asyncio
from langchain.retrievers.document_compressors import CrossEncoderReranker

class AsyncReranker:
    """异步重排序，避免阻塞主线程"""
    
    def __init__(self, reranker):
        self.reranker = reranker
    
    async def arerank(self, query, documents):
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(
            None,
            lambda: self.reranker.compress_documents(documents, query)
        )
```

## 效果对比

| 方案 | 召回率 | 精确度 | 延迟 | 成本 |
|------|--------|--------|------|------|
| 纯向量检索 | 70-80% | 60-70% | 50-100ms | 低 |
| 纯 BM25 | 60-70% | 70-80% | 30-80ms | 低 |
| 混合检索 | 85-90% | 75-85% | 100-150ms | 中 |
| 混合 + 重排序 | 90-95% | 85-95% | 200-400ms | 高 |

**建议**：
- 原型验证：纯向量检索
- 生产环境：混合检索 + 重排序
- 成本敏感：混合检索（无重排序）

## 相关页面

- [[agent/RAG/RAG关键模块解析]] — 六大模块深度解析
- [[agent/RAG/RAG参数调优手册]] — 全链路参数速查表
- [[agent/RAG/RAG评估与优化]] — 评估指标与优化策略

## 来源

- LangChain 官方文档：Ensemble Retriever, Contextual Compression
- RRF 算法论文：Cormack et al., "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods"
- BGE-Reranker 模型文档：BAAI/bge-reranker-v2-m3
- 2025 RAG 混合检索最佳实践
