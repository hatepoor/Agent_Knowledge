---
tags: [rag, nav, 学习路径, 索引, 流程]
created: 2026-08-28
updated: 2026-09-04
type: concept
---

# RAG 流程导航（RAG Pipeline Index）

> 本文是 `wiki/rag/` 目录的**入口与索引**。本目录不再按"基础/进阶/实战"分层，而是**严格按照 RAG 的一条完整落地流程**（从数据接入到评估监控的 12 步）来组织。每步一个环节目录（括号为阶段归类），全部内容按此归位、原子化分册，可嵌套子目录。该目录与 [[langchain/overview/LangChain概述|langchain]]、[[langgraph/overview/LangGraph概述|langgraph]] 平级。

## 处理流程→目录对照

```
多源数据接入(01) → 数据清洗(02) → 智能分块(03) → 混合索引构建(04) → 查询转换/改写(05)
→ 混合检索(06) → 元数据过滤(07) → 交叉编码器重排(08)
→ 上下文构建与溯源(09) → LLM生成(10)
→ 安全审核与日志(11) → 离线评估与在线监控(12)
（横切：13 工程横切 · 14 范式与前沿）
```

## 十二个环节速览

| 步 | 环节 | 收录页面 |
|----|------|----------|
| **00** 导览与基础 | 总览、端到端实例、零基础问答 | [[rag/00_导览与基础/RAG基础与演进全景|RAG基础与演进全景]] · [[rag/00_导览与基础/RAG零基础一问一答|RAG零基础一问一答]] · [[rag/00_导览与基础/LangChain_RAG端到端实现|LangChain端到端实现]] |
| **01** 数据接入与解析 | 9 页：解析器选型、各格式解析、表格、图片、元数据、增量 | [[rag/01_多源数据接入/多源数据接入|接入总览]] · [[rag/01_多源数据接入/文档解析_PDF_Word_图片与表格|解析总览]] · [[rag/01_多源数据接入/PDF解析|PDF解析]] · [[rag/01_多源数据接入/Word解析|Word解析]] · [[rag/01_多源数据接入/HTML与网页解析|HTML解析]] · [[rag/01_多源数据接入/表格提取与处理|表格提取]] · [[rag/01_多源数据接入/图片提取与描述化|图片描述化]] · [[rag/01_多源数据接入/元数据与溯源锚点|元数据锚点]] · [[rag/01_多源数据接入/增量接入与去重|增量与去重]] |
| **02** 数据清洗与规范化 | 6 页：规范化、噪声、去重、质量、脱敏 | [[rag/02_数据清洗与规范化/数据清洗与规范化|清洗总览]] · [[rag/02_数据清洗与规范化/文本规范化|文本规范化]] · [[rag/02_数据清洗与规范化/噪声剔除|噪声剔除]] · [[rag/02_数据清洗与规范化/去重策略|去重策略]] · [[rag/02_数据清洗与规范化/质量过滤|质量过滤]] · [[rag/02_数据清洗与规范化/脱敏与隐私|脱敏与隐私]] |
| **03** 智能分块 | 10 页：切分方法 × 粒度关系 × 嵌入时机 | [[rag/03_智能分块/文档分块策略|分块总览]] · [[rag/03_智能分块/固定与递归分块|固定与递归]] · [[rag/03_智能分块/结构感知分块|结构感知]] · [[rag/03_智能分块/语义分块|语义分块]] · [[rag/03_智能分块/chunk大小与overlap取值|大小与 overlap]] · [[rag/03_智能分块/特殊内容分块|特殊内容]] · [[rag/03_智能分块/上下文增强分块|上下文增强]] · [[rag/03_智能分块/分块工具选型|工具选型]] · [[rag/03_智能分块/层级父子与句子窗口检索|父子与句子窗口]] · [[rag/03_智能分块/混合索引的分块策略|混合索引切分]] |
| **04** 混合索引构建 | Embedding、向量库选型、索引类型、部署、多语言 | [[rag/04_混合索引构建/Embedding与向量检索|Embedding与向量检索]] · [[rag/04_混合索引构建/向量数据库选型指南|向量数据库选型]] · [[rag/04_混合索引构建/检索索引类型与混合索引|索引类型与混合索引]] · [[rag/04_混合索引构建/ANN近似最近邻搜索|ANN近似最近邻搜索]] · [[rag/04_混合索引构建/索引与部署流水线|索引与部署流水线]] · [[rag/04_混合索引构建/多语言与中英混排检索|多语言与中英混排]] |
| **05** 查询转换与改写 | 改写/扩展、结构化槽位映射 | [[rag/05_查询转换与改写/查询改写与扩展|查询改写与扩展]] · [[rag/05_查询转换与改写/查询结构化与槽位映射|查询结构化与槽位映射]] |
| **06** 混合检索 | 稀疏+稠密、倒排索引、RRF 融合 | [[rag/06_混合检索/混合检索|混合检索]] · [[rag/06_混合检索/BM25算法详解|BM25算法详解]] · [[rag/06_混合检索/倒排索引|倒排索引]] |
| **07** 元数据过滤 | 结构化过滤、门槛过滤 | [[rag/07_元数据过滤/元数据过滤|元数据过滤]] |
| **08** 交叉编码器重排 | Cross-Encoder 精排 | [[rag/08_交叉编码器重排/重排序与交叉编码器|重排序与交叉编码器]] |
| **09** 上下文构建与溯源 | 组装、压缩、溯源引用 | [[rag/09_上下文构建与溯源/上下文构建|上下文构建]] · [[rag/09_上下文构建与溯源/上下文压缩|上下文压缩]] · [[rag/09_上下文构建与溯源/溯源与引用|溯源与引用]] |
| **10** LLM生成 | 提示词、结构化输出、生成校验 | [[rag/10_LLM生成/LLM生成|LLM生成]] · [[rag/10_LLM生成/生成校验与防幻觉|生成校验与防幻觉]] |
| **11** 安全审核与日志 | 安全合规、内容审核、审计 | [[rag/11_安全审核与日志/安全与合规|安全与合规]] · [[rag/11_安全审核与日志/审核与审计日志|审核与审计日志]] |
| **12** 评估与监控 | 离线评估集、RAGAS、在线观测 | [[rag/12_评估与监控/离线评估集|离线评估集]] · [[rag/12_评估与监控/评估体系与RAGAS|评估体系与RAGAS]] · [[rag/12_评估与监控/在线监控与可观测性|在线监控与可观测性]] |
| **13** 工程横切 | 架构、成本、多租户、性能 | [[rag/13_工程横切/生产级RAG系统架构|生产级系统架构]] · [[rag/13_工程横切/性能与成本优化|性能与成本优化]] · [[rag/13_工程横切/多租户与大并发|多租户与大并发]] |
| **14** 范式与前沿 | Agentic RAG、图谱、多模态 | [[rag/14_范式与前沿/Agentic_RAG|Agentic_RAG]] · [[rag/14_范式与前沿/GraphRAG与知识图谱|GraphRAG]] · [[rag/14_范式与前沿/多模态与长上下文RAG|多模态与长上下文]] |

## 推荐阅读路径

- **一周入门**：00 → 01（接入）→ 03（分块）→ 04（索引）→ 06（检索）→ 09/10（生成）跑通端到端。
- **检索调优方向**：05 → 06 → 07 → 08 → 09，配合 12 的评估集迭代。
- **生产/架构方向**：11 全层 + 12 评估 + 13 工程横切。
- **研究/选型**：14 全层 + 各页"相关资料"的论文与官方文档。

## 与既有 RAG 文档的关系（不重复建设）

- [[agent/RAG/RAG核心概念与演进|agent/RAG 核心概念]]：Agent 域视角的概念页；本目录为全流程主线，二者互补。
- [[langchain/modules/langchain_retrievers/VectorStoreRetriever|langchain_retrievers 模块]]：每种 Retriever 的 API 级实战，各环节以 wikilink 引用而非重写。
- [[langgraph/tutorial/RAG_Agent|langgraph RAG_Agent]]：LangGraph 实现的 Agentic RAG 实例，14 层引用。
- [[meta/RAG_vs_Wiki|RAG vs Wiki]]：方法论讨论。

## 全局权威来源清单（各页另附具体出处）

| 类别 | 来源 | 用途 |
|------|------|------|
| 综述 | [Gao et al., RAG Survey, arXiv:2312.10997](https://arxiv.org/abs/2312.10997) | RAG 三代演进（Naive/Advanced/Modular）出处 |
| 框架 | [LangChain 官方文档](https://python.langchain.com/docs/tutorials/rag/) · [LangGraph](https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_agentic_rag/) · [LlamaIndex](https://docs.llamaindex.ai/en/stable/) | 代码第一手参考 |
| 向量库 | [Milvus](https://milvus.io/docs) · [Qdrant](https://qdrant.tech/documentation/) · [Chroma](https://docs.trychroma.com/) · [pgvector](https://github.com/pgvector/pgvector) · [Weaviate](https://weaviate.io/developers/weaviate) · [FAISS](https://github.com/facebookresearch/faiss) | 索引部署参数 |
| 检索 | [Anthropic Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) · [Cohere Rerank](https://docs.cohere.com/docs/rerank-overview) · [MTEB](https://huggingface.co/spaces/mteb/leaderboard) | 分块/检索/重排改进 |
| 评估 | [RAGAS](https://docs.ragas.io/en/stable/) · [TruLens RAG Triad](https://www.trulens.org/getting_started/core_concepts/rag_triad/) · [LangSmith](https://docs.smith.langchain.com/) | 离线评估与在线监控 |
| 校验/结构化 | [CoVe arXiv:2309.11495](https://arxiv.org/abs/2309.11495) · [LangChain Query Construction](https://blog.langchain.com/query-construction) · [Contextual Compression](https://www.langchain.com/blog/improving-document-retrieval-with-contextual-compression) | 双重校验/槽位/过滤压缩 |
| 图谱 | [微软 GraphRAG](https://microsoft.github.io/graphrag/) · [arXiv:2404.16130](https://arxiv.org/abs/2404.16130) | GraphRAG 全链路 |
| 安全 | [OWASP LLM Top 10 (2025)](https://owasp.org/www-project-top-10-for-large-language-model-applications/) | LLM08 向量弱点等威胁分类 |

## 维护约定

1. 新增主题**按流程环节**归类放入对应数字目录；归属不清的放 `13_工程横切` 或 `14_范式与前沿`，并在相关页内 wikilink。
2. 原则：**原子化**——一页讲清一件事；在同一环节下可再建子目录（文件夹套文件夹）。
3. 页面须含 YAML frontmatter（tags/created/updated/type），结论数据带出处链接与检索日期。
4. 每次增补后在根 [log.md](../../log.md) 追加 `## [YYYY-MM-DD] supplement | ...` 记录。
5. 若与 `wiki/agent/RAG` 或 `langchain_retrievers` 主题重叠，优先更新/链接已有页而非另起炉灶。

## 结构变更记录

- **2026-09-19**：**内容颗粒度重构**（01–03 环节）。原各环节为「一页装所有、每主题一段话」的清单式罗列，现按「总览页 + 专题页」展开，每个专题把一个点讲透（机制 / 实操 / 陷阱三层）。
  - **01 数据接入与解析：2 → 9 页**。总览页改为「接入总览 + 解析总览」双入口；新增 PDF解析、Word解析、HTML与网页解析、表格提取与处理、图片提取与描述化、元数据与溯源锚点、增量接入与去重。其中 Word解析 与 图片提取与描述化 基于源码级验证（`msword_backend.py` 中 `generate_picture_images` 零次出现、`prov` 为空导致无页码、`image_mode` 默认 PLACEHOLDER、spacer 自动过滤机制）。原文收录的「解析工具速查表」保留在解析总览页。
  - **02 数据清洗与规范化：1 → 6 页**。新增 文本规范化、噪声剔除、去重策略、质量过滤、脱敏与隐私。
  - **03 智能分块：3 → 10 页**。新增 固定与递归分块、结构感知分块、语义分块、chunk大小与overlap取值、特殊内容分块、上下文增强分块、分块工具选型。
  - 三环节合计 **25 页 / 约 231 KB**。全部页面通过 wikilink 完整性校验（651 条有效链接、0 条断链）。详情见 [log](../../log.md)。
- **2026-09-17**：RAGAS v0.4 时效性修正（原页使用已弃用的 v0.3 `evaluate()` API，已重写为 collections API 并补入指标组合诊断矩阵、依赖冲突说明）；新增 3 页——06 层 `倒排索引`、03 层 `混合索引的分块策略`、04 层 `多语言与中英混排检索`；01 层 `文档解析_PDF_Word_图片与表格` 补入 Docling 与解析器横向选型对比。详情见 [log](../../log.md)。
- **2026-09-04**：新增新手问答页（00 层 `RAG零基础一问一答`）与文档解析细化页（01 层 `文档解析_PDF_Word_图片与表格`）；从「基础-进阶-框架实战」分层改为本流程化 12 步组织；全部 21 页迁移/拆分/补写归位，旧分层目录已移除；新增 02 清洗、04 索引类型、01 接入、09 溯源、10 生成、11 审核日志、12 离线评估集等补缺页。详情见 [log](../../log.md)。