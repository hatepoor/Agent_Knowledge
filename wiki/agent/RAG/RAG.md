---
tags: [entity, 技术, LLM, 检索]
created: 2026-06-20
updated: 2026-06-20
type: entity
---

# RAG (检索增强生成)

## 概述

RAG（Retrieval-Augmented Generation，检索增强生成）是一种将外部知识源与大语言模型（LLM）结合的技术架构。其核心流程是：在用户提问时，先从外部文档集合中**检索**相关片段，然后将检索结果作为上下文**注入** LLM 的提示词中，最后 LLM 基于这些上下文**生成**答案。

RAG 是当前最主流的"让 LLM 利用私有/最新知识"的方案，也是 [[meta/LLM_Wiki_方法论]] 中对比的**基线模式**——wiki 模式正是为了克服 RAG 的核心局限而提出的。

## 核心流程

```
用户提问 → 查询处理 → 从文档库检索相关片段 → 注入 LLM 上下文 → LLM 生成答案
```

具体步骤：
1. **文档索引**：将文档切分为片段（chunks），通过嵌入模型转换为向量，存入向量数据库
2. **查询处理**：用户提问时，将问题转换为向量
3. **向量检索**：在向量数据库中找到与问题向量最相似的文档片段（通常返回 top-k 个）
4. **上下文注入**：将检索到的片段作为"参考资料"插入 LLM 的系统提示词
5. **答案生成**：LLM 基于参考资料和自身知识生成回答

## 常见实现

| 框架/工具 | 特点 |
|-----------|------|
| LangChain | 流行的 LLM 应用开发框架，提供完整的 RAG 管道 |
| LlamaIndex | 专注于数据连接和索引的 RAG 框架 |
| 向量数据库（Pinecone, Weaviate, Chroma） | 存储和检索文档向量 |
| OpenAI Assistants API | 内置文件搜索功能的 RAG 实现 |

## 核心局限

RAG 有三个根本性局限，正是这些局限催生了 [[meta/LLM_Wiki_方法论]] 中的 wiki 模式：

### 1. 无累积性
每次查询都从原始文档开始检索。之前问过的所有问题、得到的所有答案、做过的所有分析都不保留。一个需要综合 5 篇文档的微妙问题，每次都必须重新找到并拼接相关片段。**没有积累。**

### 2. 无综合能力
RAG 擅长找到"相关片段"，但不擅长"整合理解"。它不能跨文档建立交叉引用，不能标记矛盾，不能随着时间推移深化对一个领域的理解。

### 3. 检索精度瓶颈
- 语义检索在复杂问题上可能遗漏关键片段
- 文档切分可能破坏信息的完整性
- 检索 top-k 片段的策略限制了上下文的全面性

## 适用场景

尽管有局限，RAG 在以下场景仍然是最佳选择：
- **大规模文档集**：数千或数百万文档，wiki 模式的人工策展成本不可承受
- **快速问答**：不需要深度综合，只需要从文档中找到答案
- **动态文档**：文档频繁更新，wiki 的编译成本过高
- **企业文档问答**：FAQ、客服知识库等场景

## 相关实体
- [[tools/NotebookLM]] — Google 的 RAG 产品实现
- [[tools/ChatGPT_File_Uploads]] — OpenAI 的 RAG 产品实现
- [[tools/qmd]] — 针对 wiki 场景优化的混合搜索工具
- [[agent/orchestration/LangChain]] — 提供完整 RAG 管道的 LLM 应用框架
- [[agent/RAG/LlamaIndex]] — 专注于数据连接和索引的 RAG 框架

## 相关概念
- [[meta/RAG_vs_Wiki]] — RAG 和 Wiki 模式的详细对比
- [[meta/LLM_Wiki_方法论]] — Wiki 模式正是为克服 RAG 的局限而提出
- [[meta/综合]] — RAG 缺乏的关键能力
- [[agent/RAG/RAG核心概念与演进]] — RAG 定义、三代演进与架构图
- [[agent/RAG/RAG关键模块解析]] — 六大模块深度解析与参数表
- [[agent/RAG/RAG_LangChain完整实现]] — 从文档加载到问答 Agent 的完整代码
- [[agent/RAG/RAG参数调优手册]] — 全链路参数调优速查表
- [[agent/RAG/RAG分块策略进阶]] — 高级分块技术：Parent-Child、句子窗口、Markdown/代码/表格感知
- [[agent/RAG/RAG混合检索与重排序]] — BM25+向量混合检索、RRF 分数融合、Cross-Encoder 重排序
- [[agent/RAG/RAG评估与优化]] — RAGAS 评估框架与持续优化策略
- [[agent/RAG/RAG生产部署]] — 缓存策略、监控告警、成本控制、扩展性设计、安全性

## 来源
- [[meta/LLM_Wiki_方法论]] — 原文："Most people's experience with LLMs and documents looks like RAG"
