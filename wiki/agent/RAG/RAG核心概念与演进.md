---
tags: [concept, RAG, LLM, 检索增强生成, 架构]
type: concept
created: 2026-07-08
updated: 2026-07-08
---

# RAG 核心概念与演进

> RAG（Retrieval-Augmented Generation，检索增强生成）是一种将外部知识源与大语言模型结合的技术架构：在用户提问时，先从外部文档中**检索**相关片段，再将结果作为上下文**注入** LLM 提示词，最后由 LLM **生成**答案。

## RAG 解决的三个核心问题

| 问题 | 说明 | RAG 的应对 |
|------|------|-----------|
| **幻觉（Hallucination）** | LLM 会生成看似合理但事实上不存在的信息 | 检索到的真实文档片段为生成提供事实锚点 |
| **时效性** | 模型训练数据有截止日期，无法回答最新事件 | 外部文档库可实时更新，不受训练 cutoff 限制 |
| **私域知识** | 模型没有学过企业内部文档、个人笔记等 | 通过索引私有文档，让 LLM "知道"未训练过的内容 |

## 三代演进脉络

### 1. Naive RAG（朴素 RAG）—— 基线范式

线性流水线："索引 → 检索 → 生成"三步走，被动检索。用户提问时系统从向量库取 Top-K 片段塞入 Prompt，交给 LLM 生成。

**核心局限**：词汇匹配无法捕捉深层语义关联；检索到的块信息量不稳定；整个系统过度依赖单一检索技术，优化空间极小。

### 2. Advanced RAG（高级 RAG）—— 三阶段优化

将流水线拆分为**检索前（Pre-Retrieval）、检索（Retrieval）、检索后（Post-Retrieval）**三个可独立优化的阶段。

| 阶段 | 代表技术 |
|------|----------|
| 检索前 | 文档质量优化、索引粒度调优、查询重写（Query Rewriting） |
| 检索 | 领域内 Embedding 微调、LLM-based Embedding 模型 |
| 检索后 | 重排序（Reranking）、上下文压缩（Context Compression） |

### 3. Modular RAG（模块化 RAG）—— 当前主流范式

将检索、生成、路由、重排序等功能解耦为**独立可复用的模块**，开发者可灵活组合。

**代表技术**：混合检索（全文 + 向量）、路由模块（动态决定检索方法）、工具/API 集成。核心优势是降低系统耦合度，模块可独立替换。

### 4. 前沿趋势：Agentic RAG 与 GraphRAG（2025-2026）

**Agentic RAG**：Agent 原生架构，系统自主决定"何时查、查什么、查完怎么用"。检索不再是被动触发，而是 Agent 主动推理后的决策。

**GraphRAG**：向量数据库 + 知识图谱"双引擎"深度耦合。模糊问题用向量检索（速度快、覆盖广），复杂关系/因果推理用知识图谱（精度高、逻辑严谨），复杂推理任务的幻觉率降低约 40%。

### 演进对比

| 维度 | Naive | Advanced | Modular | Agentic/Graph |
|------|-------|----------|---------|---------------|
| 检索方式 | 单一(BM25/向量) | 调优 Embedding + 重排 | 混合 + 路由 | Agent 自主决策 + KG |
| 流水线结构 | 线性刚性 | 三阶段优化 | 解耦模块 | Agent 原生 |
| 灵活性 | 低 | 中 | 高 | 极高 |
| 工程成熟度 | 原型 | 精炼流水线 | 生产级架构 | 智能体协同 |

## 标准 RAG 数据流

```mermaid
graph LR
    subgraph 索引阶段 Indexing
        A[原始文档] --> B[文档加载<br/>Document Loaders]
        B --> C[文本分块<br/>Chunking]
        C --> D[嵌入向量<br/>Embedding]
        D --> E[(向量数据库<br/>Vector Store)]
    end

    subgraph 检索阶段 Retrieval
        F[用户提问] --> G[查询向量化<br/>Query Embedding]
        G --> H[相似度检索<br/>Similarity Search]
        E --> H
        H --> I[检索结果<br/>Top-K Chunks]
        I --> J{重排序?}
        J -->|是| K[重排序 Rerank]
        J -->|否| L[直接输出]
        K --> L
    end

    subgraph 生成阶段 Generation
        L --> M[上下文注入<br/>Context Injection]
        M --> N[Prompt 模板]
        N --> O[LLM 生成]
        O --> P[最终回答][[
    end

    style A fill:#e1f5fe
    style F fill:#fff3e0
    style P fill:#e8f5e9
    style E fill:#fce4ec
```

三阶段的核心职责：索引阶段负责将原始文档转化为可检索的向量；检索阶段负责从向量库中找到与用户问题最相关的片段；生成阶段负责将检索结果注入 Prompt 并由 LLM 生成最终回答。

## 相关页面

- [[agent/RAG/RAG]] — RAG 实体页：技术概述与常见实现
- [[agent/RAG/RAG关键模块解析]] — 六大模块的深度解析与参数表
- [[agent/RAG/RAG_LangChain完整实现]] — 从文档加载到问答 Agent 的完整代码
- [[agent/RAG/RAG参数调优手册]] — 全链路参数调优参考
- [[meta/RAG_vs_Wiki]] — RAG 与 Wiki 模式的详细对比
- [[agent/orchestration/LangChain]] — 提供完整 RAG 管道的开发框架

## 来源

- LangChain 官方文档：RAG Tutorials
- 2025-2026 RAG 技术综述（Agentic RAG, GraphRAG）
- [[meta/LLM_Wiki_方法论]] — RAG 作为对比基线
