---
tags: [concept, RAG, 模块, 文档加载, 分块, 嵌入, 向量库, 检索, 生成]
type: concept
created: 2026-07-08
updated: 2026-07-08
---

# RAG 关键模块深度解析

> 📌 **历史遗留页面**：权威内容见 [[rag/00_导览与基础/RAG基础与演进全景]]。对照见 [[agent/RAG/README|RAG 迁移对照表]]。

> RAG pipeline 由六大核心模块组成，每个模块的参数选择直接影响端到端的检索质量和生成效果。本文逐一拆解每个模块的原理、选型和参数。

## 2.1 文档加载与解析（Document Loaders & Parsing）

文档加载是 RAG 的入口，负责将 PDF、Markdown、HTML、代码等格式转换为统一的 `Document` 对象（`page_content` + `metadata`）。

### 常见加载器

| 加载器 | 导入路径 | 适用格式 | 关键参数 |
|--------|---------|---------|---------|
| `PyPDFLoader` | `langchain_community.document_loaders` | PDF | `file_path`（支持 URL）、`extract_images` |
| `DirectoryLoader` | `langchain_community.document_loaders` | 批量目录 | `glob`、`loader_cls`、`use_multithreading` |
| `UnstructuredMarkdownLoader` | `langchain_community.document_loaders` | Markdown | `mode`（`"single"` / `"elements"`） |
| `TextLoader` | `langchain_community.document_loaders` | 纯文本 | `encoding` |
| `WebBaseLoader` | `langchain_community.document_loaders` | 网页 | `web_path`、`requests_per_page` |
| `CSVLoader` | `langchain_community.document_loaders` | CSV | `source_column`、`csv_args` |

### 选型建议

- **PDF**：`PyPDFLoader` 适合文本型 PDF；扫描件需先用 OCR
- **Markdown**：`TextLoader` 适合简单加载；`UnstructuredMarkdownLoader(mode="elements")` 可拆分标题/正文/列表
- **批量目录**：`DirectoryLoader` + `glob="**/*.pdf"` 递归匹配，配合 `loader_cls` 指定加载器

## 2.2 文本分块策略（Chunking）

分块决定了检索粒度：太大稀释语义、太小丢失上下文。

### 分块器对比

| 分块器 | 导入路径 | 原理 | 适用场景 |
|--------|---------|------|---------|
| `CharacterTextSplitter` | `langchain_text_splitters` | 按单一分隔符切分 | 结构化文本（CSV、日志） |
| `RecursiveCharacterTextSplitter` | `langchain_text_splitters` | 按层级分隔符递归切分 | **通用首选**，Markdown/长文 |
| `SemanticChunker` | `langchain_experimental.text_splitter` | 按语义相似度断点切分 | 高质量分块，需嵌入模型 |

### 参数对照表

| 参数 | 所属分块器 | 默认值 | 调优范围 | 影响 | 建议 |
|------|-----------|--------|---------|------|------|
| `chunk_size` | Recursive / Character | 1000 | 200-2000 | 小→检索精准但可能丢上下文；大→上下文完整但稀释嵌入质量 | 500-1000（通用），200-500（QA） |
| `chunk_overlap` | Recursive / Character | 200 | chunk_size 的 10%-20% | 防止语义在切分边界断裂 | 50-200 |
| `separators` | Recursive | `["\n\n","\n"," ",""]` | 自定义 | 控制切分优先级 | Markdown 用 `["\n## ","\n### ","\n\n","\n"," "]` |
| `add_start_index` | Recursive | False | True/False | metadata 中记录块起始位置 | 建议开启，便于溯源 |
| `breakpoint_threshold_type` | SemanticChunker | `"percentile"` | `percentile`/`standard_deviation`/`interquartile` | 控制语义断点灵敏度 | 通用用 `percentile`，技术文档用 `standard_deviation` |
| `breakpoint_threshold_amount` | SemanticChunker | 95 | 50-95 | 值越小分块越多 | 90-95 |
| `min_chunk_size` | SemanticChunker | 无 | 50-300 | 防止过小块 | 150 |
| `max_chunk_size` | SemanticChunker | 无 | 500-2000 | 防止过大块 | 800 |

### Token 计量模式

按 token 而非字符切分，确保与 LLM 上下文窗口精确对齐：

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter
splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    encoding_name="cl100k_base",  # OpenAI 模型 tokenizer
    chunk_size=200,               # 200 tokens ≈ 约 800 字符
    chunk_overlap=50,
)
```

## 2.3 向量表示与嵌入模型（Embedding）

嵌入模型将文本转换为高维向量，是语义检索的基础。

### Dense vs Sparse Embedding

| 类型 | 原理 | 代表模型 | 特点 |
|------|------|---------|------|
| Dense（稠密） | 每个维度都有值，捕捉语义 | OpenAI text-embedding-3, BGE-M3 | 语义理解强，通用性好 |
| Sparse（稀疏） | 类似 BM25 的词频矩阵 | SPLADE, BGE-M3（稀疏模式） | 精确词匹配强，可解释性好 |

> BGE-M3 可同时输出 Dense + Sparse + Multi-Vector 三种向量，是 2025 年最全能的开源嵌入模型。

### 主流模型选型

| 模型 | 提供方 | 维度 | 最大输入 | 中文能力 | 成本 | 许可证 |
|------|--------|------|----------|----------|------|--------|
| text-embedding-3-small | OpenAI | 1536（可截断至 256） | 8191 tokens | 中等 | $0.02/1M tokens | 商业 |
| text-embedding-3-large | OpenAI | 3072（可截断） | 8191 tokens | 中等 | $0.13/1M tokens | 商业 |
| BGE-M3 | BAAI/智源 | 1024 | 8192 tokens | 极强 | 免费自建 | Apache 2.0 |
| M3E-base | Moka | 768 | 512 tokens | 极强 | 免费自建 | Apache 2.0 |
| GTE-large-en-v1.5 | Alibaba | 1024 | 512 tokens | 中等 | 免费自建 | Apache 2.0 |

### 相似度度量方式

| 度量方式 | 公式含义 | 适用场景 | 注意事项 |
|---------|---------|---------|---------|
| Cosine（余弦） | 向量夹角 | **通用首选** | 归一化后等价于点积 |
| Euclidean（欧氏） | 向量直线距离 | 图像检索 | 受向量模长影响 |
| Dot Product（点积） | 内积 | 已归一化的向量 | 未归一化时结果不可比 |

> **关键**：使用 E5、BGE 等模型时，必须为查询和文档设置特定前缀（如 `query: ` / `passage: `），否则检索质量显著下降。

## 2.4 向量数据库与索引（Vector Store & Indexing）

### 主流方案对比

| 方案 | 部署方式 | 索引类型 | 元数据过滤 | 适用规模 | 特点 |
|------|---------|---------|-----------|---------|------|
| FAISS | 本地内存 | Flat/IVF/HNSW | 需手动实现 | <1000万 | Meta 开源，纯内存，极速 |
| Chroma | 本地/嵌入 | HNSW | 支持 | <100万 | 开发友好，Python 原生 |
| Milvus | 分布式 | 多种 | 支持 | 亿级 | 生产级，云原生 |
| Qdrant | 本地/云 | HNSW | 支持 | 千万级 | Rust 实现，高性能 |
| Pinecone | SaaS | 专有 | 支持 | 亿级 | 全托管，零运维 |

### 索引类型权衡

| 索引类型 | 原理 | 速度 | 精度 | 内存 | 适用场景 |
|---------|------|------|------|------|---------|
| Flat | 暴力线性扫描 | 最慢 O(N) | 100% 精确 | 最低 | <1万向量，精度优先 |
| IVF | K-means 聚类 + 倒排表 | 快（亚线性） | 近似（调 `nprobe`） | 中 | 百万级，性价比平衡 |
| HNSW | 多层图 + 贪心搜索 | 最快 O(log n) | 高（调 `ef`） | 最高 | 在线服务，低延迟高召回 |

### 元数据过滤

```python
# Chroma 元数据过滤示例
docs = vectorstore.similarity_search(
    query="LangChain",
    k=4,
    filter={"source": "docs", "page": {"$gt": 0}},  # 支持 $eq, $ne, $gt, $in 等
)
```

## 2.5 检索策略（Retrieval）

### 策略对比

| 策略 | 核心机制 | 关键参数 | 适用场景 |
|------|---------|---------|---------|
| Similarity Search | 余弦相似度最近邻 | `k` | 通用搜索，查询明确 |
| MMR | 兼顾相关性与多样性 | `k`, `fetch_k`, `lambda_mult` | 避免重复，广覆盖 |
| Multi-Query | LLM 生成多个子查询后合并 | 无额外参数 | 提升召回率，歧义查询 |
| Parent Document | 小块检索 + 大块返回 | `parent_splitter`, `child_splitter` | 需完整上下文的长文档 |
| Rerank | 检索后用 Cross-Encoder 重排 | `top_n`, `model` | 精度要求高，生产环境 |

### MMR 参数详解

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `k` | 4 | 最终返回文档数 |
| `fetch_k` | 20 | MMR 过滤前的候选数 |
| `lambda_mult` | 0.5 | 0=最大多样性，1=最大相关性 |

### 重排序器选型

| 重排器 | 类型 | 模型 | 特点 |
|--------|------|------|------|
| CohereRerank | 商业 API | rerank-english-v3.0 / rerank-v4 | 32K 上下文，Pro/Fast 两档 |
| CrossEncoderReranker | 开源本地 | BAAI/bge-reranker-base | 免费，BGE 系列 |
| FlashrankRerank | 开源轻量 | ms-marco-MiniLM-L-12-v2 | 极速，CPU 可用 |
| JinaRerank | 商业 API | jina-reranker-v2 | 8K tokens |

## 2.6 生成增强（Generation）

### Context Injection 模板设计

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

system_prompt = (
    "你是一个专业的问答助手。请仅根据以下检索到的上下文回答问题。\n"
    "如果上下文中没有相关信息，请明确说'根据现有资料无法回答'。\n"
    "回答时请标注信息来源（如：[文档1]、[文档2]）。\n\n"
    "检索到的上下文：\n{context}"
)

qa_prompt = ChatPromptTemplate.from_messages([
    ("system", system_prompt),
    MessagesPlaceholder("chat_history"),  # 可选：多轮对话历史
    ("human", "{input}"),
])
```

### 引用溯源实现思路

1. **元数据携带**：检索结果中的 `Document.metadata` 携带 `source`、`page` 等字段
2. **Prompt 指令**：要求 LLM 在回答中标注 `[文档N]` 格式的引用
3. **后处理映射**：将 LLM 输出中的 `[文档N]` 映射回实际文档路径

```python
# 检索结果携带元数据
for i, doc in enumerate(result["context"]):
    print(f"[文档{i+1}][[ source={doc.metadata.get('source')}, page={doc.metadata.get('page')}")
```

### 无检索结果兜底策略

| 策略 | 实现 | 适用场景 |
|------|------|---------|
| 降级回答 | Prompt 中指示"如上下文不足请说明" | 通用 |
| 扩大检索 | 增加 `k` 或降低 `score_threshold` 后重试 | 检索阈值过高 |
| 查询重写 | LLM 重写查询后二次检索 | 查询表述不佳 |
| 混合检索 | 切换为 BM25 + 向量混合检索 | 词汇不匹配 |
| 拒绝回答 | 返回"无法回答"而非幻觉 | 高可靠性场景 |

## 相关页面

- [[agent/RAG/RAG核心概念与演进]] — RAG 定义、三代演进与架构图
- [[agent/RAG/RAG_LangChain完整实现]] — 完整可运行代码
- [[agent/RAG/RAG参数调优手册]] — 全链路参数速查表
- [[agent/RAG/向量数据库]] — 向量数据库实体详解
- [[langchain/modules/langchain_documents/概述]] — LangChain 文档加载模块
- [[langchain/modules/langchain_embeddings/概述]] — LangChain 嵌入模块
- [[langchain/modules/langchain_vectorstores/概述]] — LangChain 向量存储模块
- [[langchain/modules/langchain_retrievers/概述]] — LangChain 检索器模块

## 来源

- LangChain 官方文档：RAG Tutorials, Text Splitters, Retrievers
- 2025 主流 Embedding 模型评测（BGE-M3, M3E, GTE）
- 向量数据库索引原理（FAISS 文档, Milvus 文档）
