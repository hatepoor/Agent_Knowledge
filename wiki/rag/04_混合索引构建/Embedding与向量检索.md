---
tags: [rag, 基础原理, embedding, 向量检索, similarity]
created: 2026-08-28
updated: 2026-08-28
type: concept
source_url: https://python.langchain.com/docs/concepts/embedding_models/
---

# Embedding 与向量检索

## 概述

Embedding 是把文本（token/词/句子/文档）映射为**稠密低维实数向量**的技术，语义相近的文本在向量空间距离也近。向量检索就是在该空间里做最近邻搜索（ANN）取回 Top-K。它是 RAG 的"语义雷达"：召回质量的上限由 Embedding 模型 + 检索算法共同决定。

## 一、Embedding 原理要点

### 1. 从词向量到上下文向量
- 早期 Word2Vec/Glove：静态词向量，一词一义。
- 现代 Embedding 模型（BGE/OpenAI text-embedding/E5/Jina/Cohere embed v4 等）基于 Transformer 编码器，为**整段文本**产出定长向量，且随上下文变化——因此"文档块 vs 查询"需要各自向量化，查询通常较短、文档块较长。
- 双语/多语言模型（如 BGE-M3）可跨语言检索。

### 2. 相似度度量
| 度量 | 公式 | 适用 | 备注 |
|------|------|------|------|
| 余弦相似度 | `cos(A,B)=A·B/(|A||B|)` | 文本检索默认选择 | 对向量模长不敏感，需归一化后与内积等价 |
| 内积 | `A·B` | 已归一化向量 | 速度最快 |
| 欧氏距离（L2） | `sqrt(Σ(ai-bi)²)` | 归一化后与余弦排序等价 | 部分库默认 L2 |
> 一致性坑：训练/发布 Embedding 时的归一化约定要与向量库索引对齐（cosine vs L2 vs IP），否则阈值与排序含义都会变。

### 3. 维度与质量
- 常见维度：OpenAI text-embedding-3-large 3072（可降维）、BGE-M3 1024、Cohere embed-v4 1024；中文场景常用 1024/1536 档。
- 维度越高表达力强但存储/内存/延迟线性增长；text-embedding-3 支持 Matryoshka 降维（如 3072→1024），可在几乎不掉点情况下省 2/3 存储。
- **模型与查询方必须一致**：生产系统中 Embedding 模型版本是索引的"模式（schema）"，升级模型=全量重索引，见 [[rag/04_混合索引构建/索引与部署流水线|索引与部署流水线]]。

### 4. 如何选模型（2025–2026 实操口径）
**先照默认走，别在模型清单里纠结**：中文 / 多语言 + 自托管 → **BGE-M3**（1024 维，`normalize_embeddings=True` 配 cosine）；走托管 API → **OpenAI text-embedding-3-large**（支持 Matryoshka 降到 1024 省存储）。只有这两条都不满足你（如需要 8k+ 超长上下文 → jina-embeddings-v3 / voyage-3-large；需要更强跨语言或字段级召回 → E5 / gte 系）才往下看 MTEB 换模型。换模型 = 全量重建索引（见 [[rag/04_混合索引构建/索引与部署流水线|索引与部署流水线]]）。

要判断"是否该换"，再按流程验证：
1. 看 **MTEB/中文榜单**（HuggingFace 公开榜）而非厂商宣传词。
2. 小数据集上做**领域内检索评测**（Recall@k），成本极低、决策最准。
3. 私有数据安全要求高 → 本地部署开源模型；反之用托管 API。
4. 关注长文档能力：是否需要 8k+ 上下文。

## 二、向量检索与 ANN 索引

精确 KNN 复杂度 O(N×D)，百万级不可行，因此用近似最近邻（ANN）：

| 索引 | 原理 | 特点 | 场景 |
|------|------|------|------|
| Flat（暴力） | 全量计算 | 100% 召回、O(N) | 小数据集/基准验证 |
| IVF（倒排） | 聚类 + 桶内搜索，nprobe 控制 | 快、召回随 nprobe 提高 | 中大规模、内存敏感 |
| HNSW（跳表图） | 多层可导航小世界图 | 召回/速度/内存均衡，构建慢、内存高 | 生产最常用 |
| DiskANN/Vamana | 图 + SSD 分层 | 十亿级、内存友好 | 超大规模 |
| 量化（SQ/PQ） | 压缩向量位宽 | 内存省数倍、召回略降 | 内存受限大库 |

工程要点：HNSW 的 `M`（连接数，大→召回高内存高）、`efConstruction`（构建精度）、`efSearch`（查询精度，动态可调）；IVF 的 `nlist/nprobe`。索引参数是**召回-延迟-内存**三角的旋钮，不是越大越好。

## 三、LangChain 最小可运行示例

```python
# pip install -qU langchain-huggingface langchain-community chromadb
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_community.vectorstores import Chroma

# 1) Embedding 模型（示例：BGE-M3 的 HuggingFace 封装，可用本地/托管 API 替换）
embeddings = HuggingFaceEmbeddings(
    model_name="BAAI/bge-m3",           # 中文友好、1024 维、多语言
    model_kwargs={"device": "cpu"},
    encode_kwargs={"normalize_embeddings": True},  # 归一化 → 与 cosine 检索一致
)

# 2) 向量化并入库（入库时自动切块可选，此处用整句）
texts = ["RAG 通过检索为生成提供证据", "HNSW 是图式 ANN 索引", "余弦相似度衡量语义距离"]
db = Chroma.from_texts(texts, embeddings, persist_directory="./demo_db")

# 3) 向量检索 Top-K（内部：query 向量化 → ANN 搜索 → 按相似度返回）
docs = db.similarity_search_with_relevance_scores("怎么降低幻觉", k=2)
for doc, score in docs:
    print(round(score, 4), "|", doc.page_content)
```

> 规范化用法提醒：LangChain 层面看 [[langchain/modules/langchain_retrievers/VectorStoreRetriever|VectorStoreRetriever]]（三种 search_type 与阈值语义）；要控制候选集与阈值的正确姿势见该页。

## 四、常见问题

**Q1：向量检索分数阈值怎么定才不误杀/不放水？**
先采样正常查询的分数分布再定阈值；注意**不同 Embedding 模型分数尺度不同**，换模型后阈值必须重标定；重排模型分数与向量相似度分数不可混用。

**Q2：短查询 vs 长文档块相似度天然偏低怎么办？**
三招：查询改写/扩展（[[rag/05_查询转换与改写/查询改写与扩展|查询增强]]）；用支持长上下文的 Embedding；或改为"向量粗召回 + Cross-Encoder 精排"，精排对长度不对称更鲁棒。

**Q3：中文效果差，是不是该换英文模型？**
先确认用的是多语言/中文模型（BGE、text-embedding-v4、gte 系）；再查切词与编码（中文按字符切分无需分词器，但块边界别切断语义）；最后用领域数据微调（LoRA + Matryoshka 头，成本可控）。

**Q4：向量检索召回准但慢，怎么调？**
① HNSW `ef_search` 与索引 `M` 权衡；② 向量量化；③ 加标量元数据预过滤缩小候选域；④ 分片/副本扩展读吞吐。内存占用公式：`N × D × 4B × (1+索引开销系数)`。

## 五、最佳实践

1. **永远归一化 + 明确度量**，并在入库与查询两端保持一致。
2. **缓存查询向量**：同一查询改写结果重复向量化纯属浪费（配合 [[rag/13_工程横切/性能与成本优化|性能与成本优化]] 的语义缓存）。
3. **记录 embedding 模型版本**为索引元数据，避免"索引是 v1、查询是 v2"的静默错配。
4. **别只信相似度**：相似度高≠回答正确，必须走 04 层做 Faithfulness/引用校验闭环。
5. 领域数据量小（<1 万块）时优先优化**分块与重排**，收益通常大于换 Embedding 模型。

## 相关资料

- LangChain Embedding 模型概念（持续更新）— https://python.langchain.com/docs/concepts/embedding_models/
- OpenAI Embeddings 指南（text-embedding-3，Matryoshka 降维）— https://platform.openai.com/docs/guides/embeddings
- BGE-M3 模型卡 — https://huggingface.co/BAAI/bge-m3
- MTEB 公开评测榜 — https://huggingface.co/spaces/mteb/leaderboard
- FAISS 官方仓库（IVF/HNSW/PQ 实现参考）— https://github.com/facebookresearch/faiss
- Cohere Embed v4 文档 — https://docs.cohere.com/docs/embed-0

## 相关页面

- 上一页：[[rag/00_导览与基础/RAG基础与演进全景|RAG 基础与演进全景]]
- 下一页：[[rag/04_混合索引构建/向量数据库选型指南|向量数据库选型指南]]
- 语言模型侧对照：[[langchain/models/模型|langchain 模型总览]]
