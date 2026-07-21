---
tags: [langchain, retrievers, compression, llm, rag-optimization]
type: concept
created: 2026-07-07
updated: 2026-07-13
langchain_version: ">=0.3.0"
---

# ContextualCompressionRetriever

## 是什么

检索到了 5 篇文档，每篇 2000 字，prompt 塞不下？`ContextualCompressionRetriever` 在检索后、入 LLM 之前，用压缩器**提取每篇文档中最相关的片段**，丢掉废话。

```
检索 5 篇文档（每篇 2000 字）
    ↓ 压缩器
每篇只保留与 query 最相关的 2-3 句（200 字）
    ↓
拼进 prompt（1000 字，而不是 10000 字）
```

## 导入

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import (
    LLMChainExtractor,
    LLMChainFilter,
    EmbeddingsFilter,
    DocumentCompressorPipeline,
)
```

---

## 三种压缩器

| 压缩器 | 原理 | 是否需要 LLM | 成本 | 适用场景 |
|--------|------|:-----------:|------|----------|
| `LLMChainExtractor` | LLM 提取每个文档的相关片段 | ✅ | 高（N × 1 次 LLM） | 长文档、精度优先 |
| `LLMChainFilter` | LLM 判断整篇是否相关，保留 or 丢弃 | ✅ | 中（N × 1 次 LLM） | 快速筛除无关文档 |
| `EmbeddingsFilter` | 计算 query 与文档的相似度，低于阈值就丢掉 | ❌ | 低（无 LLM 调用） | 成本敏感、基础过滤 |
| `DocumentCompressorPipeline` | 串联多个压缩器 | 视组合而定 | 视组合而定 | 需要多阶段压缩 |

---

## LLMChainExtractor — 提取关键片段

```python
from langchain.retrievers.document_compressors import LLMChainExtractor
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 创建压缩器
compressor = LLMChainExtractor.from_llm(llm)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
)

# 上游先检索 5 篇，然后 LLM 提取每篇关键句
results = compression_retriever.invoke("LangChain 支持哪些功能？")

# 压缩后每篇文档的 page_content 不是全文，而是提取出的关键句
for doc in results:
    print(f"压缩后: {doc.page_content[:100]}...")  # 只有几十到几百字
```

**成本计算**：
```
每次 invoke = 1 次检索 + N × 1 次 LLM 调用
N = 检索返回的文档数
如果 k=5，每次 invoke 额外消耗 5 次 LLM 调用
```

> 用便宜的模型做压缩，用强模型做最终回答——压缩不需要推理能力，指令遵循就够了。

---

## LLMChainFilter — 整篇丢弃/保留

不修改文档内容，只判断"这篇文檔和 query 有关吗？"然后丢弃无关的：

```python
from langchain.retrievers.document_compressors import LLMChainFilter

compressor = LLMChainFilter.from_llm(llm)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
)

results = compression_retriever.invoke("LangChain 的安装方法")
# 结果可能从 5 篇变成 2-3 篇，丢弃了与安装无关的文档
# 但保留的文档内容不变（不压缩文本）
```

---

## EmbeddingsFilter — 零 LLM 成本过滤

纯基于相似度分数，不需要 LLM：

```python
from langchain.retrievers.document_compressors import EmbeddingsFilter
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

compressor = EmbeddingsFilter(
    embeddings=embeddings,
    similarity_threshold=0.75,  # 低于 0.75 的文档直接丢弃
    k=3,  # 最多保留 3 篇
)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 10}),
)

results = compression_retriever.invoke("某查询")
# 从 10 篇候选中，过滤掉相似度 < 0.75 的，再取 top-3
```

> 这是**性价比最高**的方案——0 额外 LLM 成本，非常适合预算紧张的生产环境。

---

## DocumentCompressorPipeline — 链式压缩

多个压缩器串联，分阶段处理：

```python
from langchain.retrievers.document_compressors import DocumentCompressorPipeline, EmbeddingsFilter, LLMChainExtractor

# 阶段1: 先用 Embeddings 快速筛掉不相关的
stage1 = EmbeddingsFilter(embeddings=embeddings, similarity_threshold=0.6)

# 阶段2: 再用 LLM 提取关键片段
stage2 = LLMChainExtractor.from_llm(llm)

# 组合
pipeline = DocumentCompressorPipeline(transformers=[stage1, stage2])

compression_retriever = ContextualCompressionRetriever(
    base_compressor=pipeline,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 10}),
)
# 10 → Embeddings 过滤到 5 → LLM 压缩 → 最终 3-5 篇精简文档
```

---

## 压缩器选择决策树

```
你的场景是：
├── 钱不是问题，精度最重要 → LLMChainExtractor
├── 预算有限，快速筛除无关文档   → LLMChainFilter
├── 预算紧张，不需要 LLM      → EmbeddingsFilter
└── 既要快速过滤又要精准提取   → DocumentCompressorPipeline
```

---

## 成本对比（以 k=5 为例）

| 压缩器 | 额外 LLM 调用 | 额外 token 消耗 | 延迟增加 | 推荐场景 |
|--------|:-----------:|---------------|----------|----------|
| 无压缩 | 0 | 0 | 0 | 文档都很短 |
| EmbeddingsFilter | 0 | 0 | ~50ms | **首选，0 成本** |
| LLMChainFilter | 5 次 | ~1000 | ~2-5s | 需要理解文档是否相关 |
| LLMChainExtractor | 5 次 | ~3000 | ~3-8s | 文档长，必须压缩 |

---

## 故障排查

### 问题 1：LLMChainExtractor 压缩后丢失了重要信息

**原因**：压缩器 LLM 太弱或 prompt 不匹配。

**修正**：用相同的 LLM 或能力相当的模型做压缩。

```python
# ❌ 用便宜的弱模型压缩，强模型回答
compressor = LLMChainExtractor.from_llm(cheap_llm)  # 可能丢掉关键信息

# ✅ 压缩模型和回答模型用同级别
compressor = LLMChainExtractor.from_llm(ChatOpenAI(model="gpt-4o", temperature=0))
```

### 问题 2：EmbeddingsFilter 过滤太猛，结果为空

**原因**：`similarity_threshold` 设太高。

**修正**：先在 VectorStore 上用 `similarity_search_with_score` 看实际分数分布。

### 问题 3：Pipeline 太慢

**原因**：每个阶段串行执行，总延迟 = 各阶段之和。

**修正**：减少 pipeline 阶段数；把 EmbeddingsFilter 放在 LLM 前，先用快的方式筛一轮。

---

## 相关

- [[langchain/modules/langchain_retrievers/概述|← 返回 retrievers 概述]]
- [[langchain/modules/langchain_retrievers/VectorStoreRetriever|VectorStoreRetriever]] — 被压缩的底层检索器
- [[langchain/memory/上下文压缩|上下文压缩]] — 压缩在记忆系统中的应用
