---
tags: [rag, 检索, ann, hnsw, ivf, pq, 近似最近邻, 向量检索]
created: 2026-09-10
updated: 2026-09-10
type: concept
source_url: https://qdrant.tech/documentation/concepts/indexing/
---

# ANN 近似最近邻搜索（Approximate Nearest Neighbor）

> 检索链路第 4 步。本文专讲 ANN **算法原理**（为什么快、精度换在哪）；索引选型与混合索引构建见 [[rag/04_混合索引构建/检索索引类型与混合索引|检索索引类型与混合索引]]；Embedding 上层概念见 [[rag/04_混合索引构建/Embedding与向量检索|Embedding与向量检索]]。

## 概述

向量检索的本质是"**找和查询向量最相似的 K 个文档向量**"——这叫**最近邻搜索（NN）**。当文档量到百万级以上，精确逐一比较太慢，ANN 用"**提前组织空间结构、查询时只搜可能相关的区域**"的策略，以极小的精度损失换取毫秒级查询速度。在 RAG 中，ANN 是向量检索的底层引擎，Chroma、Qdrant、FAISS、pgvector 等库默认或常用 HNSW 算法实现 ANN。

---

## 一、为什么需要 ANN

### 精确 NN 的问题

精确最近邻（暴力搜索）：拿查询向量和库里**每一个**向量逐一算距离，取最小的 K 个。

```
库里有 1000 万条文档 → 每次查询算 1000 万次距离 → 秒级延迟
```

| 文档量 | 精确 NN 延迟 | 能否用于生产 |
|--------|-------------|-------------|
| 1 万 | ~10ms | 可以 |
| 10 万 | ~100ms | 勉强 |
| 100 万 | ~1s | 不行 |
| 1000 万 | ~10s | 完全不行 |

生产 RAG 通常百万级以上文档，精确 NN 完全不可行。

### ANN 的核心思想

**用空间换时间**：提前把向量空间组织成特殊数据结构，查询时只搜索"可能相关的区域"，跳过大量无关向量。

| 维度 | 精确 NN（暴力） | 近似 ANN |
|------|-----------------|-----------|
| 速度 | O(N)，线性扫描 | O(log N) |
| 精度 | 100% | ~95-99%（可调） |
| 10M 文档查询 | 秒级 | **毫秒级** |
| 要不要建索引 | 不用 | 要建索引 |
| 用在哪 | 小数据集、实验 | **生产环境必用** |

---

## 二、四种主流 ANN 算法

### 1. HNSW（分层小世界图）—— 生产默认

**原理**：建多层"高速公路图"，从顶层粗跳、逐层细化。

```
第 3 层（最稀疏）：A ←→ B              ← 跨大步，快速逼近目标区域
  ↓
第 2 层（中等）：  A ←→ C ←→ B          ← 中步，缩小范围
  ↓
第 1 层（最密集）：A←→D←→C←→E←→B       ← 小步，精确定位
```

**类比**：地图导航——先走高速公路到城市，再走省道到街道，最后步行到门口。

**查询过程**：
1. 从最顶层入口节点开始
2. 在当前层做贪心搜索（走向离查询最近的邻居）
3. 找到当前层最近点后，下降到下一层
4. 重复直到最底层，返回 Top-K

**参数**：

| 参数 | 含义 | 默认 | 调大效果 | 调小效果 |
|------|------|------|----------|----------|
| `ef_construction` | 建索引时的搜索宽度 | 128~200 | 索引质量更高，建索引更慢 | 索引质量下降 |
| `ef_search` | 查询时的搜索宽度 | 50~100 | 召回更准，查询更慢 | 查询更快，可能漏召回 |
| `M` | 每个节点的连接数 | 16 | 图更密，内存更大 | 内存小，但可能漏路 |

**优点**：查询速度快、召回率高、不需要训练数据。
**缺点**：内存占用较高（图结构）、索引构建较慢。

**使用库**：Chroma（默认）、Qdrant（默认）、FAISS（`IndexHNSWFlat`）、pgvector（`hnsw` 索引类型）。

### 2. IVF（倒排文件 + 聚类）

**原理**：把向量空间用 K-Means 聚成很多"簇"，查询时只搜查询所在簇（及邻近簇）。

```
全部向量 → K-Means 聚成 1024 个簇
查询时：找到查询最近的 nprobe 个簇 → 只在这些簇里精确搜索
```

**类比**：翻通讯录——先翻到"M"页，再找"马"，不用翻整本。

**参数**：

| 参数 | 含义 | 默认 | 调大效果 |
|------|------|------|----------|
| `nlist` | 聚成多少个簇 | sqrt(N) | 簇更细，每簇更少，查询更快 |
| `nprobe` | 查询时搜几个簇 | 8~16 | 召回更高，但更慢 |

**优点**：内存效率好、可和 PQ 量化组合。
**缺点**：需要训练（K-Means）、簇边界可能漏召回。

**使用库**：FAISS（`IndexIVFFlat`）、Milvus。

### 3. PQ（乘积量化）

**原理**：把高维向量切成多段，每段用聚类中心编号代替，把 1024 字节的 float 向量压缩到几十字节。

```
原始向量 [1024维 float] → 切成 8 段 → 每段用 256 个中心之一编号
→ 压缩为 8 字节（压缩 128 倍）
距离计算用查表法，极快
```

**优点**：内存极省（10-100x 压缩）、距离计算快。
**缺点**：有量化误差，精度下降。

**常与 IVF 组合**：IVF + PQ = 先聚类筛区域，再用压缩码快速比距离。FAISS 的 `IndexIVFPQ` 是经典组合。

### 4. ScaNN（Google 出品）

**原理**：各向异性量化 + 剪枝，在精度和速度之间做了更优的折中。

**优点**：在很多基准测试里 SOTA。
**缺点**：生态不如 HNSW 成熟，RAG 框架集成少。

### 算法选型速查

| 你的场景 | 推荐算法 | 理由 |
|----------|----------|------|
| 通用 RAG、中等规模 | **HNSW** | 速度/召回均衡、主流默认 |
| 内存敏感、百万级 | **IVF + PQ** | 压缩省内存 |
| 极致召回要求 | **HNSW + 大 ef_search** | 调高 ef 逼近精确 |
| 小数据集（< 1万） | **Flat（暴力）** | 不建索引，100% 精确 |
| 亿级海量 | **DiskANN** | 磁盘存储，成本可控 |

---

## 三、在 RAG 中的使用方式

### 你写的这行代码底层就是 ANN

```python
results = vectorstore.similarity_search(query, k=5)
# 底层：query → embedding → ANN 搜索 → 返回 Top-5 最相似的文档向量
```

### 各向量库的 ANN 配置

**Chroma（默认 HNSW，通常不用调）**
```python
import chromadb
client = chromadb.PersistentClient(path="./chroma_db")
collection = client.create_collection("docs")
# HNSW 参数在 collection metadata 里配置
collection = client.create_collection("docs", metadata={
    "hnsw:space": "cosine",      # 距离度量：cosine / l2 / ip
    "hnsw:construction_ef": 200,  # 建索引宽度
    "hnsw:search_ef": 100,        # 查询宽度
    "hnsw:M": 16,                 # 每节点连接数
})
```

**FAISS（可选不同算法）**
```python
import faiss
import numpy as np

vectors = np.array(embeddings).astype("float32")
dim = vectors.shape[1]

# 方案1：精确（小数据集）
index = faiss.IndexFlatL2(dim)

# 方案2：HNSW（推荐）
index = faiss.IndexHNSWFlat(dim, 16)  # M=16
index.hnsw.efConstruction = 200
index.hnsw.efSearch = 100

# 方案3：IVF + PQ（大数据集省内存）
quantizer = faiss.IndexFlatL2(dim)
index = faiss.IndexIVFPQ(quantizer, dim, 1024, 8, 8)  # nlist=1024
index.train(vectors)  # IVF 需要训练
index.nprobe = 16

index.add(vectors)
D, I = index.search(query_vec.reshape(1, -1), k=5)  # 返回距离和索引
```

**pgvector（PostgreSQL 扩展）**
```sql
-- 创建向量列
CREATE TABLE docs (id serial, content text, embedding vector(1536));

-- 建 HNSW 索引
CREATE INDEX ON docs USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);

-- 查询时设置 ef_search
SET hnsw.ef_search = 100;
SELECT content FROM docs ORDER BY embedding <=> '[0.1, 0.2, ...]' LIMIT 5;
```

### 管线位置

```
文档入库                              查询时
  ↓                                    ↓
分块 → Embedding → 存入 ANN 索引    查询 → Embedding → ANN 搜索 → Top-K
                      ↑                          ↑
                  建索引（一次）            查询（每次）
```

---

## 四、ANN 精度调优实战

### 第一步：建立精确基线

```python
# 用 IndexFlatL2 跑精确 NN，作为 Recall 上限
flat_index = faiss.IndexFlatL2(dim)
flat_index.add(vectors)
_, gt = flat_index.search(query_vec.reshape(1, -1), k=10)
# gt 就是"正确答案"的索引列表
```

### 第二步：调 ANN 参数逼近基线

```python
def measure_recall(ann_index, queries, gt, k=10):
    """计算 ANN 的 Recall@k（相比精确 NN 的召回率）"""
    _, ann_result = ann_index.search(queries, k)
    recall = 0
    for i in range(len(queries)):
        gt_set = set(gt[i][:k])
        ann_set = set(ann_result[i][:k])
        recall += len(gt_set & ann_set) / k
    return recall / len(queries)

# 从小 ef 开始，逐步调大直到 Recall@10 > 0.95
for ef in [10, 25, 50, 100, 200]:
    index.hnsw.efSearch = ef
    r = measure_recall(index, test_queries, gt, k=10)
    print(f"ef_search={ef}: Recall@10={r:.3f}")
    if r > 0.95:
        break  # 够用了
```

**经验**：先调 `ef_search`（查询时的杠杆），不够再调 `M` 和 `ef_construction`（需要重建索引）。大多数场景 `ef_search=50~100` 就能达到 95%+ 召回。

---

## 五、常见问题

**Q1：ANN 和向量检索是什么关系？**
向量检索是"用向量相似度找文档"这个**任务**，ANN 是完成这个任务的**底层算法**。关系类似"搜索"和"倒排索引"——前者是需求，后者是引擎。

**Q2：ANN 的精度损失会不会影响 RAG 效果？**
会有影响但通常可忽略——95%+ 召回意味着 10 条结果里平均漏掉 0.5 条。加上后续 Rerank 精排，最终效果接近精确搜索。代价是毫秒 vs 秒级的速度差异。

**Q3：什么时候该用精确搜索而不是 ANN？**
文档量 < 1 万、或者对召回有极致要求（如法律检索不能漏）时用 Flat。其余生产场景都用 ANN。

**Q4：embedding 模型换了，ANN 索引还能用吗？**
不能。不同 embedding 模型的向量空间完全不同，换了必须**全量重建索引**。这是高频事故源。

---

## 相关导航

- 上层概念：[[rag/04_混合索引构建/Embedding与向量检索|Embedding与向量检索]]
- 索引选型：[[rag/04_混合索引构建/检索索引类型与混合索引|检索索引类型与混合索引]]
- 向量库选型：[[rag/04_混合索引构建/向量数据库选型指南|向量数据库选型指南]]
- 另一路检索：[[rag/06_混合检索/BM25算法详解|BM25算法详解]]（稀疏）
- 混合检索融合：[[rag/06_混合检索/混合检索|混合检索]]（RRF）
- 精排补偿：[[rag/08_交叉编码器重排/重排序与交叉编码器|重排序与交叉编码器]]
- 新手入口：[[rag/00_导览与基础/RAG零基础一问一答|RAG零基础一问一答]]

## 相关资料

- [Qdrant Indexing 文档](https://qdrant.tech/documentation/concepts/indexing/) — HNSW 参数详解
- [FAISS Wiki](https://github.com/facebookresearch/faiss/wiki) — IVF/PQ/HNSW 实现指南
- [pgvector Indexes](https://github.com/pgvector/pgvector#indexes) — PostgreSQL 向量索引
- [Malkov & Yashunin, Efficient and robust approximate nearest neighbor search using HNSW, 2018](https://arxiv.org/abs/1603.09320) — HNSW 原始论文
- [Johnson et al., Billion-scale similarity search with GPUs, 2019](https://arxiv.org/abs/1702.08734) — FAISS 设计论文
